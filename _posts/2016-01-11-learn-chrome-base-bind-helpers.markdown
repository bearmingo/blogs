---
layout: post
title: "Chrome base 代码解读之bind_helper"
date: 2016-01-11
categories: chrome-base
---

# SFINAE

首先介绍一种SFINAE(Substitution Failure Is Not An Error),中文叫做匹配错误不是一个编译时错误。这个由Daveed Vandevorde与Nicolai Josuttis提出，它意味着宁可对有问题类型不考虑函数的重载也不要产生产生一个错误。换句话说：编译器在辩认函数模版时，假如有一个特化会导致编译时错误（即出现编译失败），只要还有别的选择可以被选择，那么就无视这个特化错误而去选择另外的可选选择。

接入来看下[Wiki][sfinae-ref]的例子

{% highlight cpp %}
struct Test {
  typedef int foo;
};

template<typename T>
void f(typename T::foo) {}  // Definition #1

template<typename T>
void f(T) {}               // Definition #2

int main() {
  f<Test>(10);             // Call #1;
  f<int>(10);              // Call #2;
}

{% endhighlight %}

使用`int`配置模版时, #1匹配时`int`内部没有`foo`的类型定义，但程序还是正常编译了。因为还有一个#2模版定义可以使用。

封装函数有`base::Unretained`, `base::Owned()`, `base::Passed()`, `base::ConstRef()`和`base::IgnoreResult`。

`base::Unretained()`使得`Bind()`函数邦定一个不支持引用计数器的类。并且假如参数是支持引用计数的，那么这个参数的引用计数不会增加。

`base::Owned()`将一个实例的所的权传递给邦定函数返回的`Callback`, `Callback`被删除时这个实例也会被删除。


`base::Passed()`将可移动但不可拷贝的参数（例如：`scoped_ptr`）传递给`Callback`。从逻辑上来说，传递给目标函数的参数状态是具有破坏性的。即假如`Callback`创建时带有`Passed()`参数，调用`Callback::Run()`需要使用CHECK(),因为第一次调用时已经传递所有权给目录函数了。

`base::ConstRef()`邦定的是一个对参数的常量引用，而不是参数的拷贝。

`base::IgnoreResult`是将一个返回值不是`void`的函数改造成一个返回是`void`的函数。最常使用的情况是，如果有一个函数，他的返回值是个没用的`void`, 当你需要将这个函数与`PostTask`或者其它需要一个无返回值的`Callback`.

`base::Unretained`的例子：

{% highlight cpp %}
class Foo {
public:
  void func() { cout << "Foo:f" << endl; }
};

// In some function somewhere
Foo foo;
Closure foo_callback =
    Bind(&Foo::func, Unretained(&foo));
foo_callback.Run();    // Printf "Foo::f"
{% endhighlight %}

[sfinae-ref]: https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error
