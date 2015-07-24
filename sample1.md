---
layout: post
title: "Types and Programming Languages (3)"
description: ""
category:
tags: [PL, OCaml, Programming]
---
{% include JB/setup %}

## Subtyping

subtyping解决的问题是多态，OO的一个基本要素。

    we say that S is a subtype of T, written S <: T, to mean that any term of type S can safely be used in a context where a term of type T is expected. This view of subtyping is often called the principle of safe substitution.

这章只是以record来作为例子说明，直白的所一个类型S是另外一个类型的T的子类型，意思是任何使用T的context，我们可以安全的使用S。对于record类型来说，field数量多的是field数量少的子类型，因为这样任何从T要取得的field都可以从子类型里面取到。

对于函数类型来说，如果`S1->S2`, `T1->T2`, S1是T1的子类型，S2是T2的子类行，那么`S1->S2`是`T1->T2`的子类型。

引入Top类型，是所有类型的父类，对应很多编程语言里面的Object(OOP里面常见的伎俩)，Go里面我就这样定义：

{% highlight go %}
type Object interface{}
{% endhighlight %}

引入Bottom类型似乎就没什么大用处了，还增加了typecheker的复杂度。

## Ascription and Casting

类型的强制转换，分为up-cast和down-cast。up-cast对于类型检查来说要简单一些，比如类型`Animal -> Dog`, `Animal -> Cat`，由`Cat`到`Animal`的类型转换为up-cast。在很多语言里面是当做一种抽象方法。

down-cast要复杂一些，而且也可能会导致类型系统的不安全，比如：

{% highlight ocaml %}
f = λ(x:Top) (x as {a:Nat}).a;
{% endhighlight %}

这个函数接收任何类型的参数，但是隐含一个假设，必须是一个有成员变量为数字类型的a，如果传递一个错误的参数typechecker也不报错，但运行的时候就会有错误了。所以含有down-cast的类型系统应该遵循： `trust, but verify`，编译的时候不报错，但是留着运行的时候检查。为了避免down-cast引起的复杂问题，ML等语言选择的是`down-cast with type tags`。

`channels`:

    The key observation is that, from the point of view of typing, a communication channel behaves exactly like a reference cell: it can be used for both reading and writing, and, since it is difficult to determine statically which reads correspond to which writes, the only simple way to ensure type safety is to require that all the values passed along the channel must belong to the same type.


subtyping的引入导致分支多的情况下类型检查麻烦，因此引入了Join和Meet的概念，实现可参考代码里面的:

{% highlight ocaml %}
let rec join ctx tyS tyT =
  if subtype ctx tyS tyT then tyT else
  if subtype ctx tyT tyS then tyS else
  let tyS = simplifyty ctx tyS in
  let tyT = simplifyty ctx tyT in
  match (tyS,tyT) with
    (TyArr(tyS1,tyS2),TyArr(tyT1,tyT2)) ->
      TyArr(meet ctx  tyS1 tyT1, join ctx tyS2 tyT2)
  | _ ->
      TyTop

and meet ctx tyS tyT =
  .......

{% endhighlight %}

## Case Study: Imperative Objects

不考虑实现效率和语法简洁的条件下，目前为止学到的语言特性已经足够来模拟实现OOP。最简单的例子就是一个counter:

{% highlight ocaml %}
c = let x = ref 1 in
    {get = λ_:Unit. !x,
    inc = λ_:Unit. x:=succ(!x)};
{% endhighlight %}

OOP作为一种抽象手段，可以让通过接口来隐藏实现，客户端的代码只通过同一个接口才操作各种子类的对象。这里的例子一个子类只是比父类多接口而已。

{% highlight ocaml %}
newResetCounter =
    λ_:Unit. let x = ref 1 in
    {get = λ_:Unit. !x,
    inc = λ_:Unit. x:=succ(!x), reset = λ_:Unit. x:=1};
{% endhighlight %}

self的简单是现实需要动态找到对应的method，更高效的实现当然是对象创建好后method table建好。
