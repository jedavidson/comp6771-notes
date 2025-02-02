# C++ Basics

In this section, we give an overview of some assorted features of C++ on top of C, as well as some of their associated best practices.

## `auto` type inference

Try to always use `auto` to get type inference to do the heavy lifting for you.
This is a compile time cost, so there is no runtime penalty incurred for doing this.

Be careful though, as it sometimes doesn't work out:

```cpp
auto xs = std::vector<int>{1, 2, 3, 4};

// doesn't quite work, and causes an underflow error that makes this loop infinite
// i is deduced to be of type std::size_t, which is an unsigned type
for (auto i = xs.size(); i >= 0; --i) {
    // ...
}

// of course, you can help type inference out in ambiguous situations like this with type casts,
// but this is just explicitly typing the declaration with extra steps
// (one may also argue that this is a contrived example, because reverse iteration
// would be better achieved by iterators)
```

There is a doctrine known as [almost always `auto`](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/), which posits the following uniform style for declaring variables:

```cpp
auto var = type{init};

// for example,
auto y = std::string{"abc"};
auto xs = std::vector<int>{1, 2, 3, 4};
auto ys = std::set<float>{};
auto x = int{3};
```

The last of these examples is, understandably, a little controversial (i.e. it is quite verbose compared to `int x = 3`).

## Use of the `const` keyword

Everything should be `const` by default, unless there is a reason for you to change it.
This makes it clearer to the reader of the code that something shouldn't/won't be modified.

The rule for how `const` applies to a type is as follows:
> `const` applies to the thing left of it; if there is nothing on the left, then it applies to the thing right of it.

For this reason, it's convenient to favour *east `const`* over *west `const`*, as it most naturally aligns with this rule:

```cpp
auto const x = 6771; // east const (i.e. right const)
const auto x = 6771; // west const (i.e. left const)

// using east const with some pointers:
int * p;             // p is a mutable pointer to a mutable int
int const * p;       // p is a mutable pointer to a constant int
int * const p;       // p is a constant pointer to a mutable int
int const * const p; // p is a constant pointer to a constant int

// using west const:
const int * p;       // p is a mutable pointer to a constant int
int * const p;       // p is a constant pointer to a mutable int
const int * const p; // p is a constant pointer to a constant int
```

`const` also has benefits in terms of allowing some compiler optimisations, and in multithreaded situations (`const` objects are a lot easier to use in a concurrent setting).

## Value semantics

The assignment operator in C++ has **value/copy semantics**.

```cpp
auto x = std::vector<int>{1, 2, 3};

// x and y are both std::vector<int>, but y is a copy of x
// any changes to x do not manifest in y, and vice versa
auto y = x;
```

## Typecasts

Implicit casting is bad, because you never know what's going to happen:

```cpp
auto const i = 42; // int
auto d = 0.0;      // double
d = i;             // 42 gets cast to double, and this happens implicitly
```

Explicit promotions are preferred in this context, as they make your intentions when casting something clear to the compiler and others reading your code.
For such casts, prefer to use `static_cast` (i.e. the compile-time typecast):

```cpp
auto const i = 42;                     // int
auto const d = static_cast<double>(i); // i
```

There are three other typecasting operations which are somewhat less common:

- `dynamic_cast`, which allows for safe downcasting in class hierarchies (see [dynamic polymorphism](dynamic_polymorphism.md))
- `const_cast`, which allows for discarding cv-qualifiers (i.e. `const` and `volatile`)
    - This can be useful when, say, dealing with legacy APIs that are not [`const` correct](https://isocpp.org/wiki/faq/const-correctness)
- `reinterpret_cast`, which allows for bitwise reinterpretation of memory
    - This can be used to do conversions between different pointer types (i.e. like casting to and from `void *` in C), and also pointer to integral type conversions

The first of these casts is benign (though does incur some runtime safety checks), but the last two are potentially unsafe casts which are best avoided unless absolutely necessary.

## Function syntax

As of C++11, there is a new, alternative syntax for functions, called *trailing return type syntax*:

```cpp
// old
int main() {
    std::cout << "Hello World!\n";
}

// new
auto main() -> int {
    std::cout << "Hello World!\n";
}
```

The beenfit of this is that it's more consistent with the notation for lambda expressions.

## Function overloading

We can declare functions with the same name, but with different formal parameters:

```cpp
auto f() -> void {}

auto f(int x) -> int {
    return x * x;
}

auto f(double x) -> double {
    return x * x;
}

auto f(int x, int y) -> int {
    return x * y;
}

f(42);
```

When looking for the right function to use, the compiler will check the following in order:

1. Functions matching the name
2. Of those functions, those with the same number of (convertible, if necessary) arguments
3. Of those functions, the best match (in the sense that the type is much better than the others in at least 1 arg)

Overloads should be trivial (e.g. with the same behaviour, just different types).
If they are non-trivial, just name the functions differently.

## Values and references

Because C++ has value semantics, we have *references* to give us reference semantics.
These are kind of like pointers, but with some differences:

- References are an alias for another object and you can use a reference to an object interchangeably with the object itself
- Don't need to do `obj->field` for accessing elements with a reference
- Are non-null
- Are immutable, i.e. what they refer to can't change once set

```cpp
auto i = 6771;
auto& j = i; // j is a reference to i
j++;         // actually changes i
```

These by default let you read and write to the thing it references.
But you can use `const` to make read-only references:

```cpp
auto i = 6771;
auto const& j = i; // j is a const reference to i; can only read it
j++;               // not allowed
```

If a variable is declared as `const`, all of its references will be read-only:

```cpp
auto const i = 6771;
auto const& j = i; // ok, explicit
auto& k = i;       // still ok, but k will implicitly be a const ref
```

References are typically faster due to avoiding copying (particularly if the values are large in memory).

## Pass-by-value and pass-by-reference

Is pretty much the same as in C, but preferring to use references over raw pointers:

```cpp
// pass by value: values are copied into memory being used to hold formal parameters
// (so this swap function doesn't actually work!)
auto swap(int x, int y) -> void {
    auto const tmp = x;
    x = y;
    y = tmp;
}

// pass by reference: formal params are just aliases for the argument, and are
// actually being used (on reads or writes) whenever the formal params are used
// (this swap function *does* work)
auto swap(int& x, int& y) -> void {
    auto const tmp = x;
    x = y;
    y = tmp;
}
```

Pass by reference is useful when the argument has no copy operation and/or the argument is large (so we avoid a potentially expensive copy).

## Range-for

A more elegant way of looping over iterable collections:

```cpp
auto xs = std::vector<int>{1, 2, 3, 4};
for (auto const& x : xs) {
    // ...
}
```

We use `const&` because

- Most of the time, you don't want to mutate the thing you're looping over
- Working with references is (usually) faster

## Enums

Function mostly the same as they do in C and other languages:

```cpp
enum class days {
    MONDAY,
    TUESDAY,
    // ...
};

auto const mon = days::MONDAY;
```

## Unnamed functions via lambda expressions

To specify an unnamed function, we can use a lambda expression:

```cpp
[capture] (args) -> ret_type {
    body
}
```

The return type is optional to specify here.

The capture part is necessary because, by default, the lambda does not get access to its surrounding scope.
Things you want to explicitly add to the scope of the lambda should be given in a comma-separated list inside the square brackets.

A (by default `const`) copy of each captured variable is made, with value initialised to that of the variable in the outer scope at the point at which the lambda is defined:

```cpp
auto n = 6771;
auto f = [n] (int x) -> int {
    return x + n;
};
n++;
f(1); // still gives 6772 even though n has changed after defining the lambda
```

To make the local copies of captured variables mutable, we can add `mutable` after the parameter list:

```cpp
[capture] (args) mutable -> ret_type {
    body
}
```

A lambda capture can contain variables with initialisers (and these may also shadow variables of the same name from an outer scope), but to modify their values, the lambda expression must still be `mutable`:

```cpp
int i = 2;
auto f = [i = 0] (int x) mutable -> int {
    return (i++) * x;
};
f(1); // 0
f(1); // 1
i;    // 2
```

To mutate the thing being captured by the lambda after it has executed, we can capture by reference: `[&var]`.

If we want everything mentioned in the body of the lambda expression to be captured by value or reference automatically without having to list them all, use `[=]` or `[&]` as the capture expression respectively.

Lambda expresions are, under the hood, actually implemented as anonymous *functors*.
A more critical difference between a functor from a function is that functors can have state (e.g. the captured variables in these examples).

## Namespaces and aliases

Namespaces allow us to group things that belong together.
They're also used to prevent similarly-named things from clashing.

```cpp
namespace my_namespace {
auto x = 6771;
} // namespace my_namespace

// refer to this in later code as my_namespace::x;
```

It's customary to not indent the namespace block itself, but its contents have its own indentation.
(This looks silly here because the examples are so small, but it makes sense for namespaces with non-trivial contents!)

They can be nested, but we prefer top-level namespaces to multi-tier:

```cpp
namespace x {
namespace y {
auto z = 6771;
} // namespace y
} // namespace x

// refer to this in later code as x::y::z

// or we could do the following to reduce nesting, which is cleaner
namespace x::y {
auto z = 6771;
} // namespace x::y
```

They can be anonymous (i.e. unnamed), which can be used to simulate the effect of `static` functions in C. These are local to the file in which they are defined:

```cpp
namespace {
auto f(int x) -> int {
    return x + 1;
}
} // namespace

// refer to the functions in such a namespace just using their names
```

We can give namespaces new names, e.g. `namespace chrono = std::chrono;`.

We always fully-qualify things (e.g. STL containers) to avoid counterintuitive behaviour with overloading resolution.
This means that `using` directives such as `using namespace std;` are frowned upon.

In addition to namespace aliases, another alternative to shortening long types are type aliases:

```cpp
// type aliases can be local to a scope (e.g. a function or class),
// or global for a file (including files which #include it)
using map_of_maps = std::unordered_map<int, std::unordered_map<int, int>>;
```

There are some instances where `using` directives *do* make sense:

```cpp
auto print_time_fact() -> void {
    // this using directive is local only to the block scope of print_time_fact,
    // and it's appropriate to do this here, because time literals would be very,
    // very painful to use otherwise
    using namespace std::chrono_literals;
    std::cout << "There are "
              << std::chrono::seconds(6771m).count()
              << " seconds in 6771 minutes\n";
}
```
