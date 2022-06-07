# COMP6771 Lecture Notes

## C++ Basics


### `auto` type inference

Try to always use `auto` to get type inference to do the heavy lifting for you. This is a compile-time cost, so there is no runtime penalty incurred for doing this.

Be careful with using `auto` though, as it sometimes doesn't work out:
```cpp
auto xs = std::vector<int>{1, 2, 3, 4};

// Doesn't quite work because i is deduced to be of type int, but v.size() isn't (it's size_t)
for (auto i = 0; i < xs.size(); ++i) {
    // ...
}

// Kind of a contrived example, because there are better ways to loop over vectors
```

### `const` keyword

Everything should be `const` by default, unless there is a reason for you to change it. This makes it clearer to the reader of the code that something shouldn't/won't be modified.

Favour *east const* over *west const*, because it tends to make things easier to read.
```cpp
auto const x = 6771; // east const (i.e. right const)
const auto x = 6771; // west const (i.e. left const)

// Using east const with some pointers:
int * p;             // p is a mutable pointer to a mutable int
int const * p;       // p is a mutable pointer to a constant int
int * const p;       // p is a constant pointer to a mutable int
int const * const p; // p is a constant pointer to a constant int

// Using west const:
const int * p;       // p is a mutable pointer to a constant int
int * const p;       // p is a constant pointer to a mutable int
const int * const p; // p is a constant pointer to a constant int
```

`const` also has benefits in terms of allowing some compiler optimisations, and in multithreaded situations (`const` objects are a lot easier to use in a concurrent setting).

### Logical operators in C++20

We can use Python-style `and`, `or`, `not` in place of `&&`, `||` and `!` in the most recent version of C++. Where possible, use these.

### Value semantics

The assignment operator has **value/copy semantics**.
```cpp
auto x = std::vector<int>{1, 2, 3};

// x and y are both std::vector<int>, but y is a copy of x
// Any changes to x do not manifest in y, and vice versa
auto y = x;
```

### Typecasting

Implicit casting is bad, because you never know what's going to happen:
```cpp
auto const i = 42; // int
auto d = 0.0;      // double
d = i;             // 42 gets cast to double, and this happens implicit
```

Explicit promotions are preferred in this context making your intentions when casting something clear to the compiler that you're doing this, and also to the programmers when they are reading the code.
```cpp
auto const i = 42;                     // int
auto const d = static_cast<double>(i); // i
```

### Function syntax

From C++11, there is a new, alternative syntax for functions:
```cpp
// Old
int main() {
    std::cout << "Hello World!\n";
}

// New
auto main() -> int {
    std::cout << "Hello World!\n";
}
```

### Function overloading

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

Overloads should be trivial (e.g. with the same behaviour). If they are non-trivial, just name the functions differently.

### Values and references

Because C++ has value semantics, we have *references* to give us reference semantics. These are kind of like pointers, but with some differences:
- References are an alias for another object and you can use a reference to an object interchangeably with the object itself
- Don't need to do `obj->field` for accessing elements with a reference
- Are non-null
- Are immutable; what they refer to can't change once set

```cpp
auto i = 6771;
auto& j = i; // j is a reference to i
j++;         // actually changes i
```

These by default let you read and write to the thing it references. But you can use `const` to make read-only references:
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

### Pass-by-value and pass-by-reference

```cpp
// Pass by value: values are copied into memory being used to hold formal parameters
// (So this swap function doesn't actually work!)
auto swap(int x, int y) -> void {
    auto const tmp = x;
    x = y;
    y = tmp;
}

// Pass by reference: formal params are just aliases for the argument, and are
// actually being used (R or W) whenever the formal params are used
// (This swap function *does* work)
auto swap(int& x, int& y) -> void {
    auto const tmp = x;
    x = y;
    y = tmp;
}
```

PBR is useful when the argument has no copy operation and/or the argument is large (so we avoid a potentially expensive copy).

### Range-for

A more elegant way of looping over iterables:

```cpp
auto xs = std::vector<int>{1, 2, 3, 4};
for (auto const& x : xs) {
    // ...
}
```

We use `const&` because
- Most of the time you don't want to mutate the thing you're looping over
- Working with references is (usually) faster

### Enums

```cpp
enum class days {
    monday,
    tuesday,
    // ...
};

auto const mon = days::monday;
```

### Miscellaneous boredom comments

For certain data structures, there is an `emplace` and a `push/insert/...` function. The difference is that `emplace` constructs an object in-place without using copies or moves by some forwarding magic.
```cpp
auto x = std::stack<MyObj>{};
x.push(MyObj(1, 2, 3)); // creates a temporary copy of the object, then moves it into the stack
x.emplace(1, 2, 3); // forwards the arguments to the MyObj constructor to create it in the stack in-place
```

This can often be a faster way of doing it as you avoid moves/copies.

## STL Containers

TODO: Summarise Hayden lectures

## STL Iterators

TODO: Summarise Hayden lectures

## STL Algorithms

TODO: Summarise Hayden lectures

## Class Types

## Operator Overloading

## Exceptions

## Resource Management

## Smart pointers

## Templates

## Custom Iterators

## Advanced Templates

## Advanced Types

## Dynamic Polymorphism
