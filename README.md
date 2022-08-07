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

---

## Operator Overloading

In C++, all operators (like `<`, `==`, `[]`) are functions and can be overloaded to work with custom classes. Each operator is prefixed by the word `operator` (e.g. the `==` operator is `operator==` as a function). In general, however, it only makes sense to create an overload for an operator of some type if, when used with that type, it has a single, obvious meaning.

|**Type**|**Operator(s)**|**Member or friend?**|
|---|---|---|
|I/O|`>>`, `<<`|Friend|
|Arithmetic|`+`, `-`, `*`, `/`|Friend|
|Comparison|`>`, `<`, `>=`, `<=`, `==`, `!=`|Friend|
|Assignment|`=`|Member (non-`const`)|
|Compound assignment|`+=`, `-=`, `*=`, `/=`|Member (non-`const`)|
|Subscript|`[]`|Member (`const` and non-`const`)|
|Increment/decrement|`++`, `--`|Member (non-`const`)|
|Dereference|`->`, `*`|Member (non-`const`)|
|Function call|`()`|Member|

### Friendship

Making a non-member function a friend of a class allows it to access otherwise private internal member fields. This obviously breaks abstraction and should be avoided wherever possible, but it does have its uses:
- Operator overloading
- Allowing member functions of related classes (e.g. iterators) to access class internals

---

## Exceptions

Exceptions are a runtime mechanism for signifying exceptional circumstances during code exectution. Exception handling is the process of managing exceptions that are raised rather than causing the program to crash.

### Exception objects

All C++ exceptions are objects which inherit from `std::exception`. As such, we
- throw exceptions by value
- catch exceptions by `const` reference

### Exception control flow

```cpp
try {
    // some code
} catch (exception1_t const& e1) {
    // some logging
} catch (exception2_t const& e2) {
    // some more logging
} catch (...) {
    // logging for any exception other than the previous 2
}
```

### Rethrow

```cpp
try {
    try {
        // some code
    } catch (exception_t const& e) {
        // some logging

        // exception is rethrown for handling by the next try/catch layer
        throw e;
    }
} catch (exception_t const& e) {
    // some further logging
}
```

### No-throw exception safety (failure transparency)

An operation that is guaranteed to never throw an unhandled exception provides no-throw exception safety. While exceptions may occur, they are handled internally. Some examples of such operations:
- Closing files
- Freeing memory
- Move constructors and move assignments
- Trivial stack object creation

### Strong exception safety (commit or rollback)

An operation that may fail, but without leaving visible effects (e.g. no modifications to the object's state) provides strong exception safety. This is the most common type of exception safety offered by C++ functions.

To achieve this, first perform all throwing operations that don't modify internal state before doing irreversible, non-throwing operations.

### Basic exception safety (no-leak guarantee)

An operation that may fail and cause side effects, but
- respects class invariants
- does not leak resources
- corrupt data
on exception provides basic exception safety. Objects are afterward left in a valid but unspecified state, as there is no telling the extent to which the side effects of partial execution have modified things.

### No exception safety

Operations that make no guarantees regarding exceptions provide no exception safety. This is often bad C++ code and should be avoided at all costs (especially since wrapping resources and attaching lifetimes to them can give at least basic exception safety).


### `noexcept`

Functions marked as `noexcept` are understood to not throw unhandled exceptions (but doesn't prevent them from doing so). STL functions can operate more efficiently on `noexcept` functions.

---

## Resource Management

### Long lifetimes

There are three ways to make an object's lifetime outlive that of its defining scope:
- Returning out from a function via copy
- Returning out from a function via reference
    - The object itself must always outlive the reference, so references to local function variables can result in undefined behaviour!
- Returning out from a function as a heap resource

### Heap allocation via `new` and `delete`

In C++, the equivalents of `malloc` and `free` are `new` and `delete`. These call the constructors/destructors of a class to create/delete instances of that class.

```cpp
int* i = new int{6771};
delete i;

std::vector<int>* v = new std::vector<int>{1, 2, 3};
delete v;

int *l = new int[123];
delete[] l; // calls destructor on each elem first before deallocing the array
```

Since the heap is global unlike local function stacks, this means they outlive their defining scopes.

### Destructors

When a non-reference object goes out of scope, the destructor of that object is called, which frees any underlying heap resources that may be under its control. Examples of where this might be useful are
- freeing pointers
- closing open files
- releasing locks

The process during which these destructor calls are inserted at the end of a scope is called *stack unwinding*.

### Rule of 5/0

When thinking about resource management for a class, there are 5 operations to keep in mind:
- Destructor
- Copy constructor and copy assignment
- Move constructor and move assignment

The *rule of 5* states that if a class defines custom implementations any of the 5 listed operations, then it should also provide custom implementations of the other 4. The reasoning behind this is that if the default behaviour isn't sufficient for one of them, then it likely isn't sufficient for the others too. (Prior to C++11, this was the rule of 3, since copy/move assignment didn't exist then.)

The *rule of 0* states that a class requiring managed resources should either
- take full responsibility its resources by declaring and implementing all 5 operations
- declare none of these operations and rely on the default implementations by instead using types that *do* internally manage each resource (through their own implementations of the 5 operations).

```cpp
class cstring {
public:
    // rule of 5: cstring takes full responsibility over its resources
    ~cstring() { delete[] p; }
    cstring(cstring const&);
    cstring(cstring&&);
    cstring operator=(cstring const&);
    cstring operator=(cstring&&);
private:
    int* p;
}

class person {
    // rule of 0: none of the 5 operations are declared and are implicitly defaulted
    // (this is now effectively just a data class)
    cstring name;
    int age;
}
```

### Custom copy constructors

Because the default behaviour of a copy constructor is to perform member-wise shallow copies of data members, any pointer resources (e.g. heap arrays) will refer to the same object in both object copies (i.e. changes in one object to this data member reflect in the other). Even worse, during destruction, such resources will be double-freed. So, when writing a copy constructor, make sure to do deep copies:

```cpp
my_vec::operator=(my_vec const& mv)
: data_{new int[mv.size_]}
, size_{mv.size_}
{
    std::copy(mv.data_, mv.data_ + mv.size_, data_);
}
```

### Custom copy assignment operators

The copy-and-swap idiom is most useful for implementing copy assignment correctly:

```cpp
my_vec::my_vec(mv_vec const& mv) {
    // use the copy constructor to create a copy of the object,
    // then swap out its internals with this object
    my_vec(mv).swap(*this);
    return *this;
}

void my_vec::swap(my_vec& mv) {
    std::swap(data_, mv.data_);
    std::swap(size_, mv.size_);
}
```

### lvalues and rvalues

An *lvalue* is an expression that is an object reference, which is to say that it refers to an object with a defined address in memory. On the other hand, an *rvalue* is anything that isn't an lvalue. Things which are lvalues include variable names, and things that are rvalues are non-string object literals (e.g. integers) and return values of functions.

```c++
int a = 3;     //  a = lvalue, 3 = rvalue
int b = f(a);  //  b = lvalue, f(a) = rvalue
int c = b;     //  a, b = lvalue
int d = a + 1; // d = lvalue, a + 1 = rvalue
```

Loosely speaking, the STL function `std::move` can be used to turn an lvalue into an rvalue. (In reality, what is created is instead an *xvalue*, or expiring value.)

### Move constructor

Rather than creating new instances of a class by copying, we could instead construct it from the internals of another object by moving.

```c++
// std::capacity(obj, val) replaces the contents of obj by val, and returns the old value
my_vec::my_vec(my_vec&& orig) noexcept
: data_{std::exchange(orig.data_, nullptr)}
, size_{std::exchange(orig.size_, 0)}
, capacity_{std::exchange(orig.capacity_, 0)} {}
```

These should always be marked `noexcept`, as throwing move constructors are very rare, and by specifying a function as `noexcept`, the compiler can perform many optimisations to improve performance (namely, it doesn't have to worry about generating exception code).

In effect, a new object is created by "stealing" the resources of the moved object. Afterwards, the moved object should be left in a *valid but indeterminate/unspecified state*. Validity is dependent on the exact type being worked with (i.e. what constitutes a valid object of one type depends on the requirements for values of that type), however being in an unspecified state means that you cannot determine what its internal state may look like (i.e. you might not be able to say for sure what a specified member field's value will be).

It is bad practice to use a moved-from object after move construction/assignment, and often constitutes undefined behaviour.

### RAII

*Resource Acquisition Is Initialisation* is a C++ programming technique in which resources (heap objects) are encapsulated inside objects, therefore binding the life cycle of an acquired resource to the lifetime of that object. We *acquire* the resource in the constructor, and we *release* the resources in the destructor.

Every resource should be owned by either
- another resource (e.g. smart pointer, data member)
- a named rsource on the stack
- a nameless temporary variable

---

## Smart pointers

In C++11 and onwards, smart pointers are a way of wrapping unnamed heap objects (i.e. raw pointers) in named stack objects so that the lifetime of that heap object can be more easily managed (in line with RAII).

### Unique pointers (`std::unique_ptr`)

A unique pointer is an abstraction that takes *unique* ownership of a heap resource. When constructed with an initial heap object to manage, the unique pointer is the sole object who owns the managed resource, with all other access to it coming from raw pointers acting as observers. This forms the common usage pattern with unique pointers: dominion over the resource is established by the unique pointer, which is then held by some object, and any additional references to the underlying heap value are via raw observer pointers.

```cpp
// my_up now owns the heap resources associated with the string "hello"
auto up = std::make_unique<std::string>("hello");

// use make_unique when constructing them, it's better practice than the alternative
// with explicit use of `new`: prone to issues regarding use of unnamed temporaries
auto up = std::unique_ptr<std::string>(new std::string("hello"));

// we can get a raw pointer to the underlying value, a so-called observer
// p_raw will be a std::string*
auto p = up.get();

// we can access the actual value pointed to using operator* and operator->
auto v = *up;

// we can also relinquish ownership of the resource, and other stuff too
// r will now be a std::string*
auto r = up.release();
```

Since they are unique, these types of pointers are not copy-constructible/assignable (as these are explicitly `delete`d).

Once a unique pointer goes out of scope, its destructor is called. Naturally, when this destructor is called, its managed heap resource is destroyed. A consequence is that any observer pointers to a unique pointer that goes out of scope will thus point to garbage memory, so some care has to be taken when

### Shared pointers (`std::shared_ptr`)

A shared pointer is an abstraction that allows ownership of an object to be shared across multiple pointers, instead of uniquely owned by one pointer. This essentially acts as a reference-counted pointer, in that the underlying heap resource is only ever destroyed once the number of shared pointers which point to it hits 0, allowing some pointers to come and go without leading to the demise of that heap object. (This reference count is updated atomically to allow use in concurrent situations.)

A shared pointer is used in much the same way as a unique pointer, except that you obviously cannot relinquish ownership of the resource, and that we can view the active reference count for the heap resource.

```c++
// sp1 is now one of the shared pointers to the string "hello"
auto sp1 = std::make_shared<std::string>("hello");

// can access the underlying value by operator*
auto v = *sp1;

{
    // we make more shared pointers by copy-constructing another shared pointer
    // this adds 1 to the reference count of all shared pointers to that resource
    auto sp2 = sp1;
    auto sp3 = sp2;

    // we can view the reference count at any time
    auto rc = sp1.use_count(); // = sp2.use_count() = sp3.use_count() too
}

// sp2 and sp3 are now destroyed, but the heap object remains until sp1 is destroyed
```

Because not all accesses of a value necessarily need to be responsible for the object itself, we can create an analogue of "observers" by using weak pointers. These do not contribute to the reference count of that heap resource, but if usage of the resource is required (perhaps only temporarily), it must be converted to a shared pointer. Otherwise, all one can do with a unique pointer is check whether the shared pointer's managed resource is there or not.

```cpp
// wp is now a weak pointer to sp1
auto wp = std::weak_ptr<std::string>(sp1);

// if pointer is expired, then the heap resource is gone
if (wp.expired()) {
    // nothing we can do with wp now safely
}

// on the other hand, if it does exist, to get access to it, we must get a "lock" on it first
// (i.e. get a shared pointer to the resource)
else {
    auto sp = wp.lock();
    // free to use the resource safely now, and this reference will be cleaned up
    // when sp goes out of scope
}

// from a weak pointer, you can view the reference count
auto rc = wp.use_count();
```

### When to use unique and shared pointers

It's almost always the case that you want unique ownership over the object instead of shared ownership, so it makes sense to use unique pointers over shared pointers.

You should use shared pointers if
- an object has multiple owners, and it's unclear as to which one will be the longest-lived (e.g. a multithreaded situation needing access to a common resource, where it may not be known which thread exits last)
- you need safe temporary access to an object, which is not guaranteed for observing raw pointers of a unique pointer

These situations are rare in simple use cases, though.

### Partial construction

If an exception is thrown in a constructor, then only some of its subobjects (i.e. fields) are fully constructed. The C++ standard states that only destructors for these fully constructed subobjects are called, but crucially the constructor of the object itself is not called.

When working with raw pointers, this is a problem, because the destructor of a pointer does nothing, leading to memory leaks. By using smart pointers, the leak is avoided, as upon subobject destruction, the destructor of a smart pointer is called, which frees underlying memory (at least in the case of a unique pointer).

So, as a general rule of thumb, it is better to manage several wrappers around individual objects which each are responsible/have ownership over that singular resource.

---

## Templates

Polymorphism is the provision of a single interface to entities of different types. Templates are a form of static polymorphism in C++.

### Function templates

A function template is a prescription for the compiler to generate particular instances of a function varying by type. The emphasis here is on the compiler's role, unlike other languages that handle this kind of polymorphism at runtime. The process of generating these instances is called *template instantiation*.

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

// note the use of template argument deduction (?)
```

While this does slow down compilation and makes binaries larger, it is often advantageous to do this work upfront before runtime as it improves performance once the code is actually run, as there are less runtime checks incurred.

### Template specialisation

If we still wish to use a function using a single interface, but vary the behaviour slightly depending on the type, we can specialise the behaviour of that templated function.

```cpp
template<typename T>
auto f(T a) -> T {
    return a;
}

// template specialisation of f, providing a custom behaviour for T = int
// the empty template paramteter list indicates that this is a templated function
// with some compilers it may be omitted but it's good to keep it in anyway
template<>
auto f(int a) -> int {
    return a + 1;
}
```

### Type and non-type parameters

A template type parameter has unknown type and no value. A non-type parameter has known type with unknown value.

An example of how this might be useful is writing a generic procedure to find the minimum element of a `std::array`. These have fixed size which is specified as a template parameter, so we cannot simply parameterise the function over its element type.

```cpp
template <typename T, std::size_t sz>
auto min_elem(std::array<T, sz> const a) -> T {
    // find the smallest element in a and return it
}

// compiler deduces what T and sz should be from a
```

### Class templates

We can do similar things for classes as well, creating *class templates*.

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

The implementation of member functions doesn't necessarily need to be inside the class definition:

```cpp
template <typename T>
class X {
    T foo_;

public:
    X(T foo) : foo_{foo} {}

    auto get_foo() -> T;
    auto set_foo(T foo) -> void;
};

template <typename T>
auto X<T>::get_foo() -> T {
    return foo_;
}

template <typename T>
auto X<T>::set_foo(T foo) -> void {
    foo_ = foo;
}
```

You can do template specialisation on classes as well:

```cpp
template <>
class X<int> {
    // we can change the internal data members of the class just fine
    // in fact, this might be a primary driver for wanting to specialise
    // the template in the first place
    int foo_;
    int bar_;

public:
    X(int foo) : foo_{foo}, bar_{-foo} {}

    auto get_foo() -> int {
        return foo_;
    }

    auto set_foo(int foo) -> void {
        if (foo != bar_) {
            foo_ = foo;
        }
    }

    // we could add more functions here to the public interface, but it's generally
    // a good idea to not, so that they're easier to use transparently as a user
};
```

### Inclusion compilation model

Templated functions/classes must be defined in header files, because template definitions have to be known at compile time. This is in contrast to the usual link-time instantiation we use when writing non-polymorphic code that can separate the interface and implementation freely.

This can cause problems though, since it technically exposes implementation details in the interface, but also because it might make compilation a bit slower.

If in the above example the `set_foo` member function was never used, then no code is actually generated for that member function. This is called *lazy instantiation.* The same is not true for non-templated classes (i.e. if a class is non-templated and has member functions which are technically not used by anything, they are still generated anyhow).

### Static members and friends of templated classes

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

---

## Constant expressions

A `constexpr` is a variable that can be calculated at compile time (a la `#define`s in C), or a function that, if its inputs are known, can be run at compile time (and its result substituted in place of that function call).

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
// OTOH if we did specify it as a constexpr int n_fact_ce, it must
int n_fact_ce = fact_ce(10);

// not evaluated at compile time
int n_fact = fact(10);
```

This has two benefits, where applicable:
- We are offloading runtime computation to compile-time computation => faster programs
- Potential errors can be flagged at compile-time rather than at runtime, making them easier to pick up on

---

## Custom Iterators

---

## Advanced Templates

---

## Advanced Types

---

## Dynamic Polymorphism

---

## Optiver guest lecture?
