---
layout: post
title: "惰性求值和流"
description: ""
category:
tags: [PL, OCaml, Programming]
---
{% include JB/setup %}

<img src="/images/lazy-eval.png" alt="lazy-eval" class="img-center" />

## 什么是惰性求值

惰性在函数式编程语言中很常见，他的通俗解释就是一个变量或者表达式，不到必要的时候不会被eval。比如函数在传递参数的时候，参数的值可以不确定。

这种方式叫做call-by-name, 首先很明显这可能会造成一部分performance差异，如果一个表达式没有用到，那么计算出其结果是毫无意义的。而惰性求值是memoized的call-by-name, 叫做call-by-need。
从技术实现上来说，一个表达式在计算其结果之前其状态是Deferred或者Delayed的，在计算之后将其结果存储下来并修改状态为Value，之后再取就没有必要重新去计算。用一些OCaml代码来说明：
{% highlight ocaml %}

# let v = lazy (print_string "performing lazy computation\n"; sqrt 16.);;
val v : float lazy_t = <lazy>

# Lazy.force v;;
performing lazy computation

- : float = 4.
# Lazy.force v;; - : float = 4.

{% endhighlight %}
关键字lazy表示延迟计算这个表达式， Lazy.force表示求值。可以看到第一次force的时候会打印出performing...信息，后面的force就直接返回了value。

为了更好的理解这个概念，我们可以实现一把Lazy。首先定义一个lazy_state:

{% highlight ocaml %}
# type 'a lazy_state =
| Delayed of (unit -> 'a)
| Value of 'a
| Exn of exn
;;

# let create_lazy f = ref (Delayed f);;

{% endhighlight %}


这个lazy_state有三种状态，第一种就是dealyed，'a表示任何类型的value。Value表示被eval过了，并且保存下来他的值。Exn表示错误或者异常的状态。那么create_lazy就表示创建一个lazy_expression,这里的参数f可以是任何类型的函数(函数的参数类型和返回类型都可以不确定)，ref是OCaml里面的类似指针的概念。

上面例子就可以这样来写了:

{% highlight ocaml %}
# let v = create_lazy (print_string "performing lazy computation\n"; sqrt 16.);;
{% endhighlight %}

然后实现核心的force:
{% highlight ocaml %}
# let force v = match !v with
    | Value x -> x        (* 如果已经求值就直接返回value *)
    | Exn e -> raise e    (* 如果发生错误，raise错误*)
    | Delayed f ->
        try
            let x = f () in  (* 如果还未求值，eval保存下来的f *)
            v := Value x;    (* 并把结果保存下来 *)
            x
        with exn ->
            v := Exn exn;    (* 如果发生错误，保存下来 *)
    raise exn
    ;;
{% endhighlight %}

这里的!v就是取这个引用里面的值(类比C语言里面的*pointer)。然后pattern match这个lazy_state，注释里面写了每一行的操作。这里的代码很简短，最核心的意思是我们能把一个函数或者代码块保存下来，在真正需要的时候去运行这个代码块。在函数式编程里面这很常见，函数和变量一样可以自由传递。虽然看起来好不起眼，不过这会给编程带来一些深刻的影响。

## Memoization
通过上面对laziness的解释，我们可以发现这个概念的核心思想类似算法设计里面的memoization，这样在计算过程中把重复计算的过程省略掉。比如这段代码有些好玩:

{% highlight ocaml %}
let memoize f =
    let table = Hashtbl.Poly.create ()
    in (fun x ->
      match Hashtbl.find table x with
      | Some y -> y
      | None ->
        let y = f x in
        Hashtbl.add_exn table ~key:x ~data:y;
        y
     );;
{% endhighlight %}
这个函数接收任何类型的函数f，他会像一个wrapper一样给你包装一下: 给你一个table用来存储这个函数的结果，键值是你的参数x，如果发现参数是x的结果还没计算的时候，把结果算出来并存储在table里面。
这里我们又能看到函数式编程带来的好处，f是任何类型的函数(这里暂且还没处理递归)，这类问题在算法设计里面挺多的比如fibnacci，edit-distance。

在递归情况下如何处理可以看看这，这是我看过的排版最好的技术类博客[Type OCaml](http://typeocaml.com):
[Recursive Memoize & Untying the Recursive Knot](http://typeocaml.com/2015/01/25/memoize-rec-untying-the-recursive-knot/)

## Stream

有了lazy的概念之后，我们可以在编程里面表示一些看起来很数学的概念，比如一个表示所有整数的流:

{% highlight ocaml %}
type 'a stream_t = Nil | Cons of 'a * (unit -> 'a stream_t)

let rec from i = Cons (i, fun() -> from (i+1))

let hd = function
    | Nil -> failwith "hd"
    | Cons (v, _) -> v

let tl = function
    | Nil -> failwith "tl"
    | Cons (_, g) -> g()

let rec take n = function
    | Nil -> []
    | Cons (_, _)  when n = 0 -> []
    | Cons (hd, g) -> hd::take (n-1) (g())

{% endhighlight %}

Cons是把两个元素组成链表，递归函数from做的事情就是把i和一个匿名函数fun() -> from(i+1)链起来，当然匿名函数又在做类似的事情。
那么(from 1)就可以表示从1开始的所有整数了，hd是取一个流的头部，tl是取流的尾部(除头部剩下的)，take是从一个流里面取前n个元素。这可是非常的方便，还有更方便的：

{% highlight ocaml %}
let rec filter f = function
    | Nil -> Nil
    | Cons (hd, g) ->
        if f hd then Cons (hd, fun() -> filter f (g()))
        else filter f (g())
{% endhighlight %}
我们虽然只知道有这么一个流，但还是可以加一个筛选条件给他，filter函数接收筛选函数f和一个流，返回的结果就是被筛选后的流！

{% highlight ocaml %}
(* delete multiples of p from a stream *)
let sift p = filter (fun n -> n mod p <> 0)

(* sieve of Eratosthenes *)
let rec sieve = function
    | Nil -> Nil
    | Cons (p, g) ->
        let next = sift p (g()) in
        Cons (p, fun () -> sieve next)

(* primes *)
let primes = sieve (from 2)
{% endhighlight %}

所有素数就可以这么来写了，有了这个流之后要取多少就取多少。

## 其他

Haskell是纯函数式纯Lazy的实现，OCaml有imperative的部分，而且运行时不是Lazy的。相对来说我更喜欢OCaml的语法以及设计原则，FP有其好处，但imperative programming也有其益处。Lazy有其好处，但还是在用户明确需要的时候能提供就好。

部分代码引用[Real World OCaml](https://realworldocaml.org/)
