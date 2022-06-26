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

---

## STL Containers

(STL = Standard Template Library)

### Sequential containers

These organise a finite set of objects into a strict linear arrangement:
- `std::vector` is a dynamically-sized array, and the most common one to use
    - Initial capacity = #. of initial elements given
    - When the size of the vector reaches capacity, the capacity is doubled
    - Does not automatically shrink capacity; use `shrink_to_fit()` to do that
    - Provides two ways to access items:
        - `v.at[i]` does bounds checking, which is more expensive but safer
        - `v[i]` doesn't do bounds checking, which is lightweight but can lead to UB
- `std::array` is a more lightweight wrapper for a C-style fixed size array
    - Crucially, while `std::vector` places its memory in the heap, the underlying array in an `std::array` lives in the stack, which can save some system calls
    - The sheer flexibility of `std::vector` means you probably want to use that most of the time unless a specific enough situation occurs for `std::array` to be suitable
- `std::deque` is a double-ended queue
- `std::forward_list` is a singly-linked list
- `std::list` is a doubly-linked list

Most operations on these are either $O(1)$ or amortised $O(1)$. In general, the performance of the last 3 is hindered by cache locality concerns.

### Ordered associative containers

These provide fast key-based retrieval of data, but with an order on the elements:
- `std::set` is a set in the mathematical sense
- `std::multiset` is a multiset in the mathematical sense
- `std::map` is a hash table
- `std::multimap` is a hash table with non-unique keys

The ordering here is element sorted order, not insertion order. This is achieved by storing them as a sorted tree, which gives most operations on them costs of $O(\log{n})$.

### Unordered associative containers

These provide even faster key-based retrieval of data via hashing, at the cost of any guaranteed ordering of the elements:
- `std::unordered_set`
- `std::unordered_map`

The average complexity of most operations on these are $O(1)$.

### Container adapters

These restrict the functionality of an existing container to provide a different set of functionalities:
- `std::stack`
- `std::queue`
- `std::priority_queue`

When declaring container adapters, the underlying sequence container can be specified. For example, by default `std::stack` will use a `std::deque` as its underlying representation.

### Inserters and back inserters

- `std::inserter(c, it)` returns an iterator that allows for the insertion of elements at the location of `it` in the container
- `std::back_inserter(c)` returns an iterator that allows for the insertion of elements at the end of the container

### Asides

For certain data structures, there is an `emplace` and a `push/insert/...` function. The difference is that `emplace` constructs an object in-place without using copies or moves by some forwarding magic.
```cpp
auto x = std::stack<MyObj>{};
x.push(MyObj(1, 2, 3)); // creates a temporary copy of the object, then moves it into the stack
x.emplace(1, 2, 3);     // forwards the arguments to the MyObj constructor to create it in the stack in-place
```

This can often be a faster way of doing it as you avoid moves/copies.

---

## STL Iterators

An abstract interface for moving through the items in a container. The implementation of the iterator is defined by whoever is writing that iterator.

### Creating iterators

```cpp
container.begin();   // mutable iterator from the start of the container
container.cbegin();  // immutable iterator from the start of the container
container.rbegin();  // mutable reverse iterator from the end of the container
container.crbegin(); // immutable reverse iterator from the end of the container

container.end();     // mutable iterator one past the end of the container
container.cend();    // immutable iterator one past the end of the container
container.rend();    // mutable reverse iterator one before the start of the container
container.crend();   // immutable reverse iterator one before the start of the container
```

### Interacting with iterators

Iterators behave a lot like pointers and not references, so you need to use `*` to actually get the item that the iterator is pointing to at any particular time. Dereferencing an `end` iterator is undefined behaviour.

There are two functions for advancing iterators non-linearly (i.e. besides doing `++`):
- `std::advance(it, n)` will advance the iterator `n` steps
- `std::next(it, n)` will create a copy of the iterator advanced `n` steps

In situations where the length of the underlying container may not be known/stored, there is a way to calculate the "distance" between two iterators:
```cpp
std::distance(it1, it2);
```

While it makes it a bit more verbose, this approach is useful in certain situations where you may not have a *random-access iterator*. For example, taking the midpoint of two iterators:
```cpp
// Only works if the iterator given back is random-access
auto mid_it_ra = (container.begin() + container.end()) / 2;

// Works regardless, though it is a bit longer
auto mid_it = std::next(container.begin(), std::distance(container.begin(), container.end()) / 2);
```

### Types of iterators

- Input iterator: read-only iterator that can be incremented or compared for (in)equality
- Output iterator: write-only iterator that can be incremented or compared for (in)equality
- Forward iterator: like an input/output iterator but you can both read and write
- Bidirectional iterator: like a forward iterator but you can decrement too
- Random-access iterator: most general kind of iterator, which, in addition to all of the previously listed features, provides
    - relational comparisons (e.g. `it1 < it2`)
    - iterator arithmetic (e.g. `it1 + it2`)
    - subscript access to values (e.g. `it[k]`, which is just `*(it + k)`)

In general read/write situations, we have forward iterators $\subset$ bidirectional iterators $\subset$ random-access iterators.

Different STL containers provide different iterator types:
- `std::vector`, `std::deque` and `std::array` give random-access iterators
- `std::list`, `std::(multi)set` and `std::(multi)map` give bidirectional iterators
- `std::forward_list`, `std::unordered_(multi)set` and `std::unordered_(multi)map` give forward iterators

Container adapters do not provide iterators.

In C++20, there are also contiguous iterators, which are random iterators that guarantee contiguity of the underlying elements.

### Asides

When using a `const` iterator in a loop for example, you don't make the actual iterator variable `const` explicitly:
```cpp
for (auto const it = c.begin(); it != c.end(); ++it) {
    // Doesn't work, needs to be auto it = ...
}
```

When working with something like a `map`, if you want to look up whether an item is in the map and then use that item if it does exist, it is often quicker to work with the `.find()` method, which will give back an iterator:
```cpp
// Iterator method: one lookup and can access the item at that key via the iterator
// Note: the iterator is to a key-value std::pair
auto it = map.find(key);
if (it != map.end()) {
    auto v = *iter.second;
}

// C++20 way of doing it via .contains() and .at(): two lookups
if (map.contains(key)) {
    auto v = map.at(key);
}
```

---

## STL Algorithms

The `<algorithm>` library gives some nice standardised functions for common tasks to cut down on boilerplate code that the programmer has to write.

These algorithms work on iterators to containers rather than containers directly so as to be most portable with different containers.

### Map and reduce equivalents

```cpp
// This is map, which places the mapped contents into a destination container
// It is up to the programmer to make sure that destination container is big enough
std::transform(src_begin, src_end, dest_begin, func);

// This is basically non-returning map
// The func should be void, because its return type is ignored
// To mutate, make sure func takes references in the argument, and change via assignment
std::for_each(begin, end, func);

// This is reduce
// Function takes accumulator, value as arguments (in that order)
std::accumulate(begin, end, initial_value, func);
```

For functional-esque tools, see the `<functional>` header (similar to Python's operator, functools).

### Lambda functions

To specify an unnamed function, we can do
```cpp
[capture] (args) -> ret_type {
    body
}
```

The capture part is necessary because, by default, the lambda does not get access to its surrounding scope. Things you want to explicitly add to the scope of the lambda should be given in a comma-separated list inside the square brackets.


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

To mutate the thing being captured by the lambda after it has executed, we can capture by reference: `[&var]`.

### Asides

Lambdas are actually anonymous *functors*. A more critical difference from functions is that functors can have state (e.g. the capture in this example).

---

## Class Types

### Construction

```cpp
// Default construction: calls the no-arg constructor
std::vector<int> v11;
auto v12 = std::vector<int>{};
auto v13 = std::vector<int>();

// Copy constructors
auto v2 = std::vector<int>{v11.begin(), v11.end()}; // == a copy of v1
auto v3 = std::vector<int>{v2};                     // == a copy of v2

// Initialiser list constructor
auto v4 = std::vector<int>{5, 2}; // == [5, 2]

// Count + value constructor
auto v5 = std::vector<int>(5, 2); // == [2, 2, 2, 2, 2]
```

Use braces to construct things unless it's necessary to use parens. This is called *uniform initialisation*.

### Namespaces

Namespaces allow us to group things that belong together. They're also used to prevent similarly-named things from clashing.

```cpp
namespace my_namespace {
    auto x = 6771;
} // namespace my_namespace

// refer to this in later code as my_namespace::x;
```

Can be nested, but we prefer top-level namespaces to multi-tier.

```cpp
namespace x {
    namespace y {
        auto z = 6771;
    } // namespace y
} // namespace x

// refer to this in later code as x::y::z

// or we could do the following
namespace x::y {
    auto z = 6771;
} // namespace x::y
```

Can have anon namespaces, to simulate the effect of static functions. These are local to the file in which they are defined.

```cpp
namespace {
    auto z = 6771;
} // namespace
```

Can give them new names, e.g. `namespace chrono = std::chrono`.

We always fully-qualify things to avoid counterintuitive behaviour with overloading resolution.

### OOP

```cpp
class foo {
// memebrs accessible by everyone
public:
    foo();

// members accessible by members, friends and subclasses
protected:
    int x;

// members accessible by members and friends
private:
    int y;
    void z();

// can have multiple sections of the same name
public:
    int w;
}
```

By default, members of a class are *private*. The only difference between a struct and a class is that all members of a struct are public by default. We will almost always use classes over structs unless we essentially just want a data class with complete access.

There is a notion of `this`, which in a method call is always a pointer to the class object which called it. However we prefer to just suffix private/internal members with an underscore.

### Class scope

It's common to declare a class with its method signatures in a header file, then implement those methods in another file. However, the implementation must scope the class when writing those implementations:

```cpp
// in Foo.h
class Foo {
public:
    Foo();
    ~Foo();
    void f();
}
```
```cpp
// in Foo.cpp
#include "Foo.h"

Foo::Foo() {
    // ...
}

Foo::~Foo() {
    // ...
}

void Foo::f() {
    // ...
}
```

### Constructors

Constructors may specify an *initialiser list*, which gives values to data members in order of their member declaration in the class itself. Crucially, this happens *before* the constructor body is actually executed.

```cpp
class MyClass {
public:
    MyClass(int i, std::vector<int> j) : i_{i}, j_{j} {} // e.g. MyClass{1, std::vector<int>{1, 2}};
private:
    int i_;
    std::vector<int> j_;
}
```

When initialising an object, the following order is used:
```
for each data member in declaration order
    if it has a used definition initialiser
        initialise it using the used defined initialiser
    else if it is of a built-in type
        do nothing (leave it as whatever)
    else
        initialise it using its default constructor
```

In other words, initialisation happens for all data members before the body is called, making so-called uniform initialisation more efficient than setting things in the constructor body (since those values are initialised first anyway, perhaps just to default values). You may as well use an initialiser list to just make those initial values meaningful.

### Delegating constructors

Constructors can call other constructors, possibly to set default values:

```cpp
class MyClass {
public:
    MyClass(int i, std::vector<int> j) : i_{i}, j_{j} {}
    MyClass(std::vector<int> j) : MyClass(6771, j) {}; // e.g. MyClass(std::vector<int>{1, 2});
private:
    int i_;
    std::vector<int> j_;
}
```

### Destructors

Methods that are called when an object goes out of scope, which can be useful for cleaning up used resources (e.g. any pointers, opened files, locks, ...). They should not throw exceptions.

```cpp
class MyClass {
    ~MyClass() noexcept;
}

MyClass::~MyClass() noexcept {
    // do destruction
}
```

### Explicit initialisation

By default, unary constructors can do implicit type conversion from the parameter to the class:
```cpp
class Age {
public:
    Age(int age): age_{age} {}
private:
    int age_;
}

// explicit construction
Age a1{12};
auto a2 = age{12};

// implicit construction
Age a = 12;
```

Sometimes you want this, other times you don't, because implicit type conversions are generally not liked. To prevent this and force people to do the explicit way, use the `explicit` keyword:
```cpp
class Age {
public:
    explicit Age(int age): age_{age} {}
private:
    int age_;
}

// explicit construction still works
Age a1{12};
auto a2 = age{12};

// implicit construction now no longer works
// Age a = 12;
```

### `const` objects and member functions

By default, member functions are only callable by non-`const` objects. Only member functions marked with `const` at the end of the function may be called on `const` objects:

```cpp
class Person {
public:
    person(std::string const& name) : name_{name} {}

    // only callable by non-const Person objects
    auto set_name(std::string const& name) -> void {
        name_ = name;
    }

    // callable by all objects
    // it is a compiler error to modify members in such a function that are not
    // declared as mutable; it's rare to want to ever make members mutable,
    // but they do have their use cases sometimes (e.g. a cache)
    auto get_name() const -> std::string const& {
        return name_;
    }
private:
    mutable int age;
    std::string name_;
}
```

### Static data members and member functions

Belong to every instance of a class:

```cpp
class MyClass {
public:
    static std::string const x;
    static void f();
}

// member function may be called as MyClass::f()
```

Static data members in general can't be initialised in the class itself, but must be done elsewhere:
```cpp
auto MyClass::x = "abcd";
```

### Special member functions, `default` and `delete`

By default, the compiler will *synthesise* or create some *special member functions* for you. Two examples are
- If no constructors are given, then a default no-arg constructor will be created for you.
- A copy constructor that allows one to construct an object as a copy of an existing one.

To signal to the compiler that you do want its synthesised constructors (e.g. the default constructor), use the keyword `default`. To signal to the compiler that you *don't* want one of its synthesised constructors (e.g. the copy constructor), use the `delete` keyword.
```cpp
class MyClass {
public:
    MyClass() = default; // generates the default constructor
    MyClass(int i) i_{i} {}
    MyClass(MyClass const& mc) = delete; // don't generate the copy constructor
private:
    int i_;
}
```

## Operator Overloading

## Exceptions

## Resource Management

## Smart pointers

## Templates

## Custom Iterators

## Advanced Templates

## Advanced Types

## Dynamic Polymorphism
