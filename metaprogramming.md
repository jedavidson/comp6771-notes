# Templates II: metaprogramming

In this section, we discuss C++ metaprogramming.
[TODO: explain a little bit more about what that is]

## Constant expressions

A *constant expression* is a variable that can be calculated at compile time (as `#define`'d values are in C), or a function that, if its inputs are known, can be run at compile time (and its result substituted in place of that function call).
We use the `constexpr` keyword to denote such things:

```cpp
constexpr int fact_ce(int n) {
    return n <= 1 ? 1 : n * fact_ce(n - 1);
}

int fact(int n) {
    return n <= 1 ? 1 : n * fact(n - 1);
}

// can easily be calculated at compile time
constexpr int n = 10 + 20;

// function call is evaluated at compile time
// the omission of the constexpr keyword here means that the compiler is allowed
// to turn this into a constant expression if it wants to, but doesn't have to
// OTOH if we did specify it as a constexpr int n_fact_ce, the compiler must do it
int n_fact_ce = fact_ce(10);

// not evaluated at compile time, because fact isn't marked constexpr
int n_fact = fact(10);
```

This has two benefits, where applicable:

- We are offloading runtime computation to compile time computation, so get faster programs
- Potential errors can be flagged at compile time rather than at runtime, making them easier to pick up on

However, the natural downside is that this workload slows compilation.

[TODO: also talk about `consteval` and `constinit`?]

## Type traits

Type traits are a mechanism to introspect about the properties of types in C++, which can be helpful when you're working with templated types.
These traits either allow you to ask questions about the type or make transformations to types (e.g. adding/removing `const`).

Traits that ask questions about types include things like

- `std::numeric_limits<T>`, which provides the minimum and maximum values of a type
    - This is in contrast to the C way to do this via `#define`s
- `std::is_signed<T>`, which provides a way to tell whether a type is signed (e.g. signed integer) or not

The "answer" to the question will be in some field of the trait (and the traits themselves usually take the form of a `struct`).

"Question traits" can be used to do conditional compilation, in combination with constant expressions:

```cpp
auto algorithm_signed(int i) -> void;
auto algorithm_unsigned(unsigned u) -> void;

template <typename T>
auto algorithm(T t) -> void {
    // provided that the conditional expression is a bool constant expression itself,
    // if constexpr can resolve which conditional branch to use at compile time
    // in this case, we keep the algorithm for the appropriate signedness of T, and
    // throw a static (compile time!) error if this isn't possible
    if constexpr(std::is_signed<T>::value) {
        algorithm_signed(t);
    }
    else if constexpr(std::is_unsigned<T>::value) {
        algorithm_unsigned(t);
    }
    else {
        static_assert(std::is_signed<T>::value || std::is_unsigned<T>::value, "must be signed or unsigned");
    }
}
```

Traits used for type transformations include things like `std::move`, which under the hood uses a type trait (called `std::remove_reference`) to perform a conversion to an rvalue reference.

## Concepts

[TODO: concepts]

## Extension: SFINAE

[TODO: substitution failure is not an error]

## Extension: CRTP

[TODO: the curiously recurring template pattern]
