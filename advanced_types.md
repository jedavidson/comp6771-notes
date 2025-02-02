# Advanced types

In this section, we discuss some more advanced aspects of C++'s type system.

## Algebraic types

[TODO: `std::optional`, `std::variant` and maybe some other stuff]

## `decltype`

Like many other languages, C++ allows you to determine at compile time what the type of an expression is. Unlike other languages (e.g. Python), this is purely a type-level mechanism, and does not allow for things such as

```python
if type(a) == int:
    # do something
```

Rather, it is intended to be used for things such as

```cpp
auto compare = [] (my_class a, my_class b) {
    // ...
};
std::priority_queue<my_class, std::vector<my_class>, decltype(compare)> pq(compare);
```

There are some rules for how this works, given an occurrence of `decltype(e)`:

1. If `e` is a variable in local or namespace scope, a static member variable or a function parameter, then the result is the variable/parameter's type `T`
2. If `e` is an lvalue (e.g. a reference), then the result is `T&`
3. If `e` is an xvalue (e.g. an rvalue reference returned by `std::move()`), then the result is `T&&`
4. If `e` is a prvalue (e.g. an integer literal), then the result is `T`

This mechanism can be useful for determining return types of templated code:

```cpp
template <typename T, typename U>
auto add(T& lhs, U& rhs) -> decltype(lhs + rhs) {
    return lhs + rhs;
}
```

A trailing return type is crucial here, since

```cpp
template <typename lhsT, typename rhsT>
decltype(lhs + rhs) add(lhsT& lhs, rhsT& rhs) {
    return lhs + rhs;
}
```

would not compile (as `lhs` and `rhs` have been used before they are declared).

## Binding

This relates to what kind of references may be bound to what kind of function arguments:

|Value type / Does it bind to this argument type?|`T&`|`T const&`|`T&&`|`template T&&`|
|--------------|---|---|---|---|
|lvalue        |Yes|Yes|   |Yes|
|`const` lvalue|   |Yes|   |Yes|
|rvalue        |   |Yes|Yes|Yes|
|`const` rvalue|   |Yes|   |Yes|

Everything binds to `const` lvalue references (since we are in essence requesting a read-only view of some value). This includes rvalues too (which are "immutable")! Similarly, everything binds to templated rvalue references.

Non-`const` lvalue and rvalues only bind to references of that same type (i.e. a non-`const` lvalue only binds to a non-`const` lvalue reference), because it is a compile error to drop the `const` qualifier of a type.

## Forwarding references (or universal references)

If a variable or parameter is declared to have type `T&&` for some deduced type `T`, that variable or parameter is called a *forwarding reference*. The requirement that type deduction is involved is critical, because everything binds to universal references (i.e. the `template T&&` from before). If there is a non-directly deduced type, then it is instead an rvalue reference.

```cpp
int n;

// lvalue reference
int& lvalue = n;

// no deduced parameter type => rvalue reference
int&& rvalue = std::move(n);

// deduced parameter type => universal reference
template <typename T> T&& universal = n;

// also a universal reference!
auto&& universal_auto = n;

// param is of universal reference type
template<typename T>
void f(T&& param);

// param1 and param2 are rvalue references, because type deduction has to be
// directly involved here
// (so saying it has to be `T&&` for deduced type `T` is strict!)
template<typename T>
void f(std::vector<T>&& param1, T const&& param2);
```

## Reference collapsing

There are rules around determining what types like `T& &&`, called the *reference collapsing rules*:
- rvalue references to rvalue references become rvalue references (i.e. `T&& &&` -> `T &`)
- All other references of references collapse to lvalue references
    - `T& &` -> `T &` (lvalue ref of lvalue ref)
    - `T&& &` -> `T &` (lvalue ref of rvalue ref)
    - `T& &&` -> `T &` (rvalue ref of lvalue ref)

## Forwarding

When writing a templated wrapper function, some problems arise:

```cpp
// fails to compile if T is non-copyable (i.e. its copy ctor/assn is deleted)
// also not very good if values of type T are expensive to copy
template <typename T>
auto wrapper(T v) -> auto {
    return fn(v);
}

// not very useful if fn needs to modify v
// also doesn't behave as intended if we pass an rvalue via std::move
// since wrapper(std::move(v)) will call fn(v) instead of fn(std::move(v))
template <typename T>
auto wrapper(T const& v) -> auto {
    return fn(v);
}

// fails if we pass it a const lvalue reference
// fails to compile if given an rvalue
template <typename T>
auto wrapper(T& v) -> auto {
    return fn(v);
}

// doesn't fail to compile anymore, but v is not actually an rvalue inside
// the function, but rather an lvalue (so fn is passed an lvalue)
// this is because named rvalue references are lvalues
template <typename T>
auto wrapper(T&& v) -> auto {
    return fn(v);
}
```

So that lvalues are treated as lvalues and rvalues are treated as rvalues in this context, we could `static_cast` on `v` before calling it, but there is a more readable alternative in `std::forward`:

```cpp
template <typename T>
auto wrapper(T&& v) -> auto {
    return fn(std::forward<T>(v)); // =~ return fn(static_cast<T&&>(v));
}
```

Prime examples of where this might be useful is in something like `make_unique`, which takes in arguments for constructing a `unique_ptr<T>` value:

```cpp
template <typename T, typename... Ts>
auto make_unique(Ts&&... args) -> std::unique_ptr<T> {
    // note that the ... is outside the forward call, and not right next to args,
    // because we want to call
    // new T(forward(arg1), forward(arg2), ...)
    // and not
    // new T(forward(arg1, arg2, ...))
    return std::unique_ptr(new T(std::forward<Ts>(args)...));
}
```

By doing this, value categories are preserved even though the arguments to `make_unique` may be a mix of lvalues and rvalues. This is called *perfect forwarding.*

You should use `std::forward` when you want to wrap functions with a parameterised type, which you might want to do for a few reasons (e.g. doing something special before/after the function call).
