# Templates I: fundamentals

In this section, we introduce the fundamental mechanics of templates, a form of *static* polymorphism in C++ (i.e. generics/compile time polymorphism).

## Function templates

A function template is a prescription for the compiler to generate particular instances of a function varying by type.
The emphasis here is on the compiler's role: this happens at compile time.
The process of generating these instances is called *template instantiation* (or *monomorphisation* in other languages).

```cpp
// T is a template type parameter
// the thing inside the <> is called a template parameter list
template<typename T>
auto min(T a, T b) -> T {
    return a < b ? a : b;
}

// because we are calling this templated function with T = int and T = double,
// the compiler will generate instances of the function min for these two types
// in the sense that there are two separate implementations present:
min(1, 2);     // auto min(int a, int b) -> int
min(0.9, 2.3); // auto min(double a, double b) -> double

// note the use of template argument deduction: we didn't write out min<T> at the call site!
// this is something the compiler can do for us, and it's good to lean on it if possible
```

While this does slow down compilation and makes binaries larger, it is often advantageous to do this work upfront before runtime as it improves performance once the code is actually run, as there are less runtime checks incurred.

## Type and non-type parameters

A template type parameter has unknown type and no value.
A non-type parameter has known type with unknown value.

An example of how this might be useful is writing a generic procedure to find the minimum element of a `std::array`.
These have fixed size which is specified as a template parameter, so we cannot simply parameterise the function over its element type.

```cpp
template <typename T, std::size_t sz>
auto min_elem(std::array<T, sz> const a) -> T {
    // find the smallest element in a and return it
}

// compiler deduces what T and sz should be from a
```

## Class templates

We can do similar things for classes as well, creating *class templates*:

```cpp
template <typename T>
class X {
    T foo_;

public:
    X(T foo) : foo_{foo} {}

    auto get_foo() -> T {
        return foo_;
    }

    auto set_foo(T foo) -> void {
    }
};

// use like this
auto x1 = X<int>{1};
auto x2 = X<std::string>{"hi"};
```

## Inclusion compilation model

Templated functions/classes *must* be defined in header files, because template definitions have to be known at compile time.
This is in contrast to the usual link-time instantiation we use when writing non-polymorphic code that can separate the interface and implementation freely.

This can cause problems though, since it technically exposes implementation details in the interface, but also because it might make compilation a bit slower.

If in the above example the `set_foo` member function was never used, then no code is actually generated for that member function.
This is called *lazy instantiation*.
The same is not true for non-templated classes (i.e. if a class is non-templated and has member functions which are technically not used by anything, they are still generated anyhow).

## Static members and friends of templated classes

Each template instantiation of a class has its own set of static members, as well as friend functions.

```cpp
template <typename T>
class X {
    T foo_;

    // each X<T>, X<U>, X<V> instantiation has its own bar_ member
    static int bar_;

public:
    X(T foo) : foo_{foo} {}

    auto get_foo() -> T {
        return foo_;
    }

    auto set_foo(T foo) -> void {
    }

    // each X<T>, X<U>, X<V> instantiation has its own operator<< friend
    friend auto operator<<(std::ostream& os, X const& x) -> std::ostream& {
        // ...
    }
};
```

## Default members

We can set defaults for template type parameters:

```c++
// if cont_t is not actually specified when invoking an instance of this templated class,
// then its default value will be std::vector<T>
template <typename T, typename cont_t = std::vector<T>>
class stack {
public:
    // interface here ...
private:
    cont_t stack_;
};
```

Now when instantiating `stack`, one can give just the element type and fall back on a container type of `std::vector` if that fits the use case.

An example of this being done in practice is `std::vector`, which can take a second type parameter for a custom element allocator. Since it is uncommon to want to do this, it has a sane default template type parameter set in this case.

All template parameter lists (e.g. in member function definitions placed outside of the class declaration) have to be updated to conform as well if a template parameter is given a default value:

```cpp
template <typename T, typename U = int>
class X {
public:
    auto f() -> T;
    auto g() -> T;
};

template <typename T, typename U>
auto X::f() -> T {
    // ...
}

template <typename T, typename U>
auto X::g() -> T {
    // ...
}

```

Template type parameters with defaults have to be placed at the end of the template parameter list, so it is not valid to start a template like

```cpp
template <typename X, typename Y = int, typename Z>
```

## Specialisation

If we want to give a more specific implementation of a templated type, we can use *specialisation* in two ways:

- *Partial specialisation* for a template for a type "based on" a template type parameter (e.g. a specialisation of a template for `T*` or `std::vector<T>`)
- *Explicit specialisation* for a fully-realised type (e.g. `std::string`, `int`)

Specialising a template is a good idea if:

- You need to preserve the existing semantics of a template for something that wouldn't otherwise work with the default generic implementation
    - Specialising to give completely different semantics/break assumptions about the behaviour of a class to other realised type parameters is poor form
- You're writing a type trait (see [metaprogramming](metaprogramming.md))
- There is an optimisation to be had with specialisation (e.g. `std::vector<bool>` is fully specialised to improve space efficiency)

It is a bad idea to specialise functions, because they cannot be partially specialised, and explicit specialisation is better done via overloading.
For this reason, explicit specialisation is only to be done on templated classes.

```cpp
// partial specialisation
// note that we've given a partial amount of info about what the template type is,
// namely that it's a pointer, hence the qualifier "partial"
template <typename T>
class stack<T*> {
public:
    // some interface here

    auto sum() -> int {
        // here instead of summing by value naively, we would probably write
        // an implementation that dereferenced each value in the std::vector
        // this would make much more sense than summing by address (!!!)
    }

private:
    std::vector<T*> stack_;
};

// explicit specialisation
// note the use of an empty template parameter list
template <>
class vector<bool> {
public:
    // regular old interface

private:
    // here instead of storing a bool[], we might use some other space-efficient
    // representation, since bools occupy only 1 bit of memory instead of the 8
    // which are packed into a byte
}
```

(This `std::vector<bool>` specialisation actually exists in C++, but has [come to be seen as a bit of a mistake in retrospect](https://stackoverflow.com/questions/17794569/why-isnt-vectorbool-a-stl-container).)

## Variadic templates

Template parameter lists can be of a variable length:

```cpp
// this acts as a "base case"
template<typename T>
T sum(T v) {
    return v;
}

// this acts as a "recursive case"
// typename... Ts is called a template parameter pack
// Ts... vs is called a function parameter pack
template<typename T, typename... Ts>
T sum(T v, Ts... vs) {
    // vs contains the parameter list less one value, so we are doing some proper
    // recursion here, although keep in mind this is all happening at compile time
    return v + sum(vs...);
}
```

We can go quite far with this and replicate pattern matching if we liked:

```cpp
template<typename T>
bool pairwise_cmp(T v1) {
  return false;
}

template<typename T>
bool pairwise_cmp(T v1, T v2) {
    return v1 == v2;
}

// here, we can actually "capture" the first 2 values instead of just 1,
// and then do variadic template "recursion" as per usual
// because we might be given an odd number of arguments though, we either
// get a compile time warning if no single-arg base case templated function
// is given, or are forced to implement one with a sensible behaviour
// (perhaps always returning false if we're pairwise comparing an odd #. of elements)
template<typename T, typename... Ts>
bool pairwise_sum(T v1, T v2, Ts... vs) {
    return v1 == v2 && pairwise_cmp(vs...);
}
```

This can also be extended to variadic templated classes: see [tuple](https://github.com/eliben/code-for-blog/blob/master/2014/variadic-tuple.cpp) as an example.

```cpp
// must wrap chain in a struct to allow partial template specialization
template <int i, class F>
struct multi {
    static F chain(F f) {
        return f * multi<i - 1, F>::chain(f);
    }
};

template <class F>
struct multi<2, F> {
    static F chain(F f) {
        return f * f;
    }
};

template <int i, class F>
F compose(F f) {
    return multi<i, F>::chain(f);
}

// this prints out 10
auto increment = std::bind(std::plus<>(), std::placeholders::_1, 1);
std::cout << compose<10>(increment)(0) << "\n";
```

## Member templates

If we wanted to support conversion between one templated class to another templated class (i.e. conversion of a stack of `int`s to a stack of `double`s), then we can use member templates to achieve this:

```cpp
template <typename T>
class stack {
public:
    // this is a member function that is itself templated by some other type
    template <typename U>
    stack(stack<U>&);

    // other stuff here

private:
    std::vector<T> stack_;
};

// when giving the definition of the class like this, the extra template type must be
// treated as an "inner" templated type
// so this would not be the same as template <typename T, typename U>
template <typename T>
template <typename U>
stack<T>::stack(stack<U>& s) {
    while (!s.empty()) {
        stack_.push_back(static_cast<T>(s.pop()));
    }
}
```

## Template template parameters

Template parameters may themselves be templates (e.g. `std::vector`) as opposed to fully-realised types.

```cpp
// be very careful when specifying the template parameter list of template template parameters
// if we were trying to use std::vector here, it technically takes in two template params
// but we can use variadic template template parameters (!!!) to fix this
// we at least want one template arg for cont_t though (the type)
template <typename T, template <typename, typename...> typename cont_t>
class stack {
    // ...
private:
    cont_t<T> container_;
};
```

This allows us to write things like

```cpp
// make it implicit that the vector has ints
auto s = stack<int, std::vector>{};
```

instead of

```cpp
// must explicitly state that the vector has ints - blergh
auto s = stack<int, std::vector<int>>{};
```

<!-- TODO: fix this -->
<!-- One thing to note here is that the compiler will only consider primary class templates while looking for a match for the template template parameter. A consequence is that partially-specialised templates ... -->

## Template argument deduction

The process by which the compiler determines what the types of type parameters and values of non-type parameters should be from the function arguments being used with them.

```cpp
template <typename T, std::size_t sz>
auto min_elem(std::array<T, sz> a) -> T {
    T min = a[0];
    for (auto i = 0; i < sz; ++i) {
        min = min < a[i] ? min : a[i];
    }
    return min;
}

// deduces that T = int, sz = 4
auto min = min_elem(std::array<int, 4>{1, 2, 3, 4});
```

This works for variadic templates too.

Template argument deduction means that it is technically not necessary to write things like

```cpp
std::vector<int>{1, 2, 3};
```

when the compiler could quite easily deduce that in

```cpp
std::vector{1, 2, 3};
```

each element is of type `int`. This is called *implicit deduction.*
But specifying the type (as in *explicit deduction*) can be used to leave the compiler in no doubt as to what the template values should be.

[TODO: extend with class template argument deduction too]
