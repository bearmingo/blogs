---
layout: post
title: "Chrome base 代码解读之tuple"
date: 2016-01-09
categories: chrome-base
---

base中自己实现tuple，没有直接使用std::tuple。tuple主要用来保存函数的参数，在DispatchToMethod函数中将保存在tuple中的数据取传给method。

tuple中使用了C++11的新特性"Variadic Template", 中文叫"可变参数模版"。可变参数模版可以带任意个数的任何类型的模版参数。模版函数与模版类都可以使用。下面是可变参数类模版的一个例子。如果想了解更多可变参数模版可以参数[C++ - New features - Variadic templats][variadic-template-ref]

{% highlight cpp %}
template<typename... Arguments>
class VariadicTemplate;
{% endhighlight %}


{% highlight cpp %}
// Index sequences
//
// Minimal clone of the similarly-named C++14 functionality.
template <size_t...>
struct IndexSequence {};

template <size_t... Ns>
struct MakeIndexSequenceImpl;
{% endhighlight %}
由于C++11还不支持参数序号的功能，所以这里使用这种模版技术模拟。

{% highlight cpp %}

#if defined(_PREFAST_) && defined(OS_WIN)
// Work around VC++ 2013 /analyze internal compiler error:
// https://connect.microsoft.com/VisualStudio/feedback/details/1053626

template <> struct MakeIndexSequenceImpl<0> {
  using Type = IndexSequence<>;
};
template <> struct MakeIndexSequenceImpl<1> {
  using Type = IndexSequence<0>;
};
...
#else  // defined(WIN) && defined(_PREFAST_)
template <size_t... Ns>
struct MakeIndexSequenceImpl<0, Ns...> {
  using Type = IndexSequence<Ns...>;
};

template <size_t N, size_t... Ns>
struct MakeIndexSequenceImpl<N, Ns...>
    : MakeIndexSequenceImpl<N - 1, N - 1, Ns...> {};
    #endif  // defined(WIN) && defined(_PREFAST_)

template <size_t N>
using MakeIndexSequence = typename MakeIndexSequenceImpl<N>::Type;
{% endhighlight %}
前面的宏定义中说在MSVS 2013中编译器不支持部分C++11的功能，所以这里以这种形式代替。宏定义else是种标准的形式。

举个例子`MakeIndexSequence<N>`的N为3, 即`MakeIndexSequenceImpl<3>::Type`. 在MSVS2013下对应的直接就是IndexSequence<0, 1, 2>。在其它编译器中先匹配到`MakeIndexSequenceImpl<N, Ns...>`这个模版，第一个模版参数N为3，第二参数Ns...为空。由于`MakeIndexSequenceImpl<N, Ns...>`又是继承`MakeIndexSequenceImpl<N - 1, N -1, Ns...>`,即`MakeIndexSequenceIndexImpl<2, 2>`。依此类推`MakeIndexSequenceIndexImpl<1, 1, 2>`, `MakeIndexSequenceIndexImpl<0, 1, 2>`， `MakeIndexSequenceImpl<0, 0, 1, 2>`。这时模版匹配到`MakeIndexSequenceImpl<0, Ns...>`,这里的Ns为`0, 1, 2`, 即Type为`IndexSequence<0, 1, 2>`。最终`MakeIndexSequenceImpl<3>::Type`是`IndexSequence<0, 1, 3>`。

{% highlight cpp %}
// Traits ----------------------------------------------------------------------
//
// A simple traits class for tuple arguments.
//
// ValueType: the bare, nonref version of a type (same as the type for nonrefs).
// RefType: the ref version of a type (same as the type for refs).
// ParamType: what type to pass to functions (refs should not be constified).

template <class P>
struct TupleTraits {
  typedef P ValueType;
  typedef P& RefType;
  typedef const P& ParamType;
};

template <class P>
struct TupleTraits<P&> {
  typedef P ValueType;
  typedef P& RefType;
  typedef P& ParamType;
};
{% endhighlight %}

上面的代码定义了Tuple参数的一此类型。第一个模版是普通类型的类型定义。假如P的类型是`int`, 匹配的模版是`struct TupleTraits`, 在tuple使用值的类型是`int`, 引用类型为`int&`, 参数类型是`const int&`。第二个模版对于P为引用的类型定义。假如P的类型是int&, 匹配到的模版是`struct TupleTraits<P&>`, 那么tuple使用的值的类型`int`, 引用的类型是`int&`, 参数的类型是`int&`。

{% highlight cpp %}
// Tuple -----------------------------------------------------------------------
//
// This set of classes is useful for bundling 0 or more heterogeneous data types
// into a single variable.  The advantage of this is that it greatly simplifies
// function objects that need to take an arbitrary number of parameters; see
// RunnableMethod and IPC::MessageWithTuple.
//
// Tuple<> is supplied to act as a 'void' type.  It can be used, for example,
// when dispatching to a function that accepts no arguments (see the
// Dispatchers below).
// Tuple<A> is rarely useful.  One such use is when A is non-const ref that you
// want filled by the dispatchee, and the tuple is merely a container for that
// output (a "tier").  See MakeRefTuple and its usages.

template <typename IxSeq, typename... Ts>
struct TupleBaseImpl;
template <typename... Ts>
using TupleBase = TupleBaseImpl<MakeIndexSequence<sizeof...(Ts)>, Ts...>;
template <size_t N, typename T>
struct TupleLeaf;

template <typename... Ts>
struct Tuple final : TupleBase<Ts...> {
  Tuple() : TupleBase<Ts...>() {}
  explicit Tuple(typename TupleTraits<Ts>::ParamType... args)
      : TupleBase<Ts...>(args...) {}
};

// Avoids ambiguity between Tuple's two constructors.
template <>
struct Tuple<> final {};

template <size_t... Ns, typename... Ts>
struct TupleBaseImpl<IndexSequence<Ns...>, Ts...> : TupleLeaf<Ns, Ts>... {
  TupleBaseImpl() : TupleLeaf<Ns, Ts>()... {}
  explicit TupleBaseImpl(typename TupleTraits<Ts>::ParamType... args)
      : TupleLeaf<Ns, Ts>(args)... {}
};

template <size_t N, typename T>
struct TupleLeaf {
  TupleLeaf() {}
  explicit TupleLeaf(typename TupleTraits<T>::ParamType x) : x(x) {}

  T& get() { return x; }
  const T& get() const { return x; }

  T x;
};
{% endhighlight %}

举个例子说明以上代码的工作原理。假如`Tuple<int, double&>`，编译推导过程如下。
1. Tuple的模版参数是`int, double`, Tuple的父类是TupleBase<int, double>，即`TupleBaseImpl<MakeIndexSequence<2>, int, double&>`。由上面得出`MakeIndexSequence<2>`就是`IndexSequence<0, 1>`,那么就变成了`TupleBaseImpl<IndexSequence<0, 1>, int, double&>`。

2. 再根据TupleBaseImpl的定义。`TupleBaseImpl<IndexSequence<Ns...>, Ts...>`中的Ns...为`0, 1`, Ts...为`int, double&`。那么TupleLeaf<Ns, Ts>...就是`TupleLeaf<0, int>`, `TupleLeaf<1, double&>`。`TupleBaseImpl`继承`TupleLeaf<0, int>`，`TupleLeaf<1, double&>`。

3. `TupleLeaf`中有一个成员变量`T x`，由模版参数决定这个成员变量的类型。`TupleLeaf<0, int>`的成员变量x的类型是`int`, `TupleLeaf<1, double&>`的成员变量类型是`double&`。

由上过程可以看出，`Tuple`可以根据参数个数的多少，继承相应个数的`TupleLeaf`, 并将数据分别存在相应的`TupleLeaf`中。

再看下`Tuple`的带参数构造函数，使用的是`TupleTraits<Ts>::ParamType`。所以上面例子中的构造函数是`Tuple(const int&, double&)`


{% highlight cpp %}
// Tuple getters --------------------------------------------------------------
//
// Allows accessing an arbitrary tuple element by index.
//
// Example usage:
//   base::Tuple<int, double> t2;
//   base::get<0>(t2) = 42;
//   base::get<1>(t2) = 3.14;

template <size_t I, typename T>
T& get(TupleLeaf<I, T>& leaf) {
  return leaf.get();
}

template <size_t I, typename T>
const T& get(const TupleLeaf<I, T>& leaf) {
  return leaf.get();
}
{% endhighlight %}
两个get模版是用来获取`Tuple`中第I个数据的引用。当变量的类型是`Tuple`(或者`Tuple&`)时，调用`get`匹配的是第一个。变量类型是`const Tuple`(或者`const Tuple&`)时，调用`get`匹配到的是第二个。

{% highlight cpp %}
// Tuple types ----------------------------------------------------------------
//
// Allows for selection of ValueTuple/RefTuple/ParamTuple without needing the
// definitions of class types the tuple takes as parameters.

template <typename T>
struct TupleTypes;

template <typename... Ts>
struct TupleTypes<Tuple<Ts...>> {
  using ValueTuple = Tuple<typename TupleTraits<Ts>::ValueType...>;
  using RefTuple = Tuple<typename TupleTraits<Ts>::RefType...>;
  using ParamTuple = Tuple<typename TupleTraits<Ts>::ParamType...>;
};
{% endhighlight %}

这个定义Tuple中所有参数的值类型/引用类型/参数类型，方便获取。

{% highlight cpp %}
// Tuple creators -------------------------------------------------------------
//
// Helper functions for constructing tuples while inferring the template
// argument types.

template <typename... Ts>
inline Tuple<Ts...> MakeTuple(const Ts&... arg) {
  return Tuple<Ts...>(arg...);
}

// The following set of helpers make what Boost refers to as "Tiers" - a tuple
// of references.

template <typename... Ts>
inline Tuple<Ts&...> MakeRefTuple(Ts&... arg) {
  return Tuple<Ts&...>(arg...);
}
{% endhighlight %}

`MakeTuple`与`MakeRefTuple`函数生成`Tuple`主要区别在与，前者的类型的输入参数的类型相同，后都都是对输入参数的引用。

{% highlight cpp %}
// Dispatchers ----------------------------------------------------------------
//
// Helper functions that call the given method on an object, with the unpacked
// tuple arguments.  Notice that they all have the same number of arguments,
// so you need only write:
//   DispatchToMethod(object, &Object::method, args);
// This is very useful for templated dispatchers, since they don't need to know
// what type |args| is.

// Non-Static Dispatchers with no out params.

template <typename ObjT, typename Method, typename... Ts, size_t... Ns>
inline void DispatchToMethodImpl(ObjT* obj,
                                 Method method,
                                 const Tuple<Ts...>& arg,
                                 IndexSequence<Ns...>) {
  (obj->*method)(base::internal::UnwrapTraits<Ts>::Unwrap(get<Ns>(arg))...);
}

template <typename ObjT, typename Method, typename... Ts>
inline void DispatchToMethod(ObjT* obj,
                             Method method,
                             const Tuple<Ts...>& arg) {
  DispatchToMethodImpl(obj, method, arg, MakeIndexSequence<sizeof...(Ts)>());
}

// Static Dispatchers with no out params.

template <typename Function, typename... Ts, size_t... Ns>
inline void DispatchToFunctionImpl(Function function,
                                   const Tuple<Ts...>& arg,
                                   IndexSequence<Ns...>) {
  (*function)(base::internal::UnwrapTraits<Ts>::Unwrap(get<Ns>(arg))...);
}

template <typename Function, typename... Ts>
inline void DispatchToFunction(Function function, const Tuple<Ts...>& arg) {
  DispatchToFunctionImpl(function, arg, MakeIndexSequence<sizeof...(Ts)>());
}

// Dispatchers with out parameters.

template <typename ObjT,
          typename Method,
          typename... InTs,
          typename... OutTs,
          size_t... InNs,
          size_t... OutNs>
inline void DispatchToMethodImpl(ObjT* obj,
                                 Method method,
                                 const Tuple<InTs...>& in,
                                 Tuple<OutTs...>* out,
                                 IndexSequence<InNs...>,
                                 IndexSequence<OutNs...>) {
  (obj->*method)(base::internal::UnwrapTraits<InTs>::Unwrap(get<InNs>(in))...,
                 &get<OutNs>(*out)...);
}

template <typename ObjT, typename Method, typename... InTs, typename... OutTs>
inline void DispatchToMethod(ObjT* obj,
                             Method method,
                             const Tuple<InTs...>& in,
                             Tuple<OutTs...>* out) {
  DispatchToMethodImpl(obj, method, in, out,
                       MakeIndexSequence<sizeof...(InTs)>(),
                       MakeIndexSequence<sizeof...(OutTs)>());
}
{% endhighlight %}

`DispatchToMethodImpl`与`DispatchToFunctionImpl`功能是使用`Tuple`中的数据作为参数调用输入的函数。这里也用到了`MakeIndexSequence`生成一个IndexSequence。

[variadic-template-ref]: http://www.cplusplus.com/articles/EhvU7k9E/
