---
title: Golang中的interface
date: 2021-10-31
categories:
tags:
  - Golang
---

在 Go 中，关键字 interface 被赋予了多种不同的含义。每个类型都有接口，意味着对那个类型定义了方法集合 。如下这段代码定义了具有一个字段和两个方法的结构类型 S。
type S struct { i int }func (p *S) Get() int { return p.i }func (p *S) Put(v int) { p.i = v }也可以定义接口类型，仅仅是方法的集合。这里定义了一个有两个方法的接口 I：

type I interface {Get() intPut(int) }
对于接口 I，S 是合法的实现，因为它定义了 I 所需的两个方法。注意，即便是没有明确定义 S 实现了 I，这也是正确的。
Go 程序可以利用这个特点来实现接口的另一个含义，就是 接口值:
func f(p I) {  //定义一个函数接受一个接口类型作为参数fmt.Println(p.Get()) //p实现了接口I，必须有get()方法p.Put(1)  //Put()方法类似}这里的变量 p 保存了接口类型的值。因为 S 实现了 I，可以调用 f 向其传递 S 类型的值的指针：

var s S ; f(&s)
获取 s 的地址，而不是 S 的值的原因，是因为在 s 的指针上定义了方法。
在 Go 中的接口有着与许多其他编程语言类似的思路：C++ 中的纯抽象虚基类，Haskell中的 typeclasses 或者 Python 中的 duck typing。然而没有其他任何一个语言联合了接口值、静态类型检查、运行时动态转换，以及无须明确定义类型适配一个接口。这些给 Go 带来的结果是，强大、灵活、高效和容易编写的。
定义另外一个类型同样实现了接口 I：

type R struct { i int }func (p *R) Get() int { return p.i }func (p *R) Put(v int) { p.i = v }
函数 f 现在可以接受类型为 R 或 S 的变量。假设需要在函数 f 中知道实际的类型。在Go 中可以使用 type switch 得到。
type switch:
func f(p I) {switch t := p.(type) { case *S: case *R: case S: case R: default:     } }在 switch 之外使用 (type) 是非法的。类型判断不是唯一的运行时得到类型的方法。
为了在运行时得到类型，同样可以使用 “comma, ok” 来判断一个接口类型是否实现了某个特定接口：

if t, ok := something.(I) ; ok {// 对于某些实现了接口 I 的// t 是其所拥有的类型}
确定一个变量实现了某个接口，可以使用：

t := something.(I)
