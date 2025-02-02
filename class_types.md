# Class Types

In this section, we give an introduction to the static (i.e. compile-time) aspects of class types and objects in C++.

## Object construction and initialisation

A class can typically offer more than one way of creating a new instance:

```cpp
// default construction: calls the constructor with no arguments
std::vector<int> v11;          // v11 == empty vector
auto v12 = std::vector<int>{}; // v12 == empty vector

// copy constructors
auto v2 = std::vector<int>{v11.begin(), v11.end()}; // v2 == copy of v11
auto v3 = std::vector<int>{v2};                     // v3 == copy of v2

// initialiser list constructor
auto v4 = std::vector<int>{5, 2}; // == vector with contents [5, 2]

// count + value constructor
auto v5 = std::vector<int>(5, 2); // == vector with contents [2, 2, 2, 2, 2]
```

Note that, aside from the last example, braces `{}` were used instead of parentheses `()` when invoking the constructor.
This is called *uniform initialisation*, which is preferred to use over *direct initialisation*, since it is more strict with narrowing conversions (e.g. `double` to `int`) that may implicitly happen with constructor arguments.

## Structure of a C++ class type

```cpp
class foo {
    // until an accessibility modifier is encountered, everything in a class is assumed private
    int a;
    void f();

// members accessible by everyone
public:
    foo();

// members accessible by members, friends and subclasses
protected:
    int x;

// members accessible by members and friends
private:
    int y;
    void z();

// can have multiple sections of the same kind
public:
    int w;
}

struct foo {
    // like a class, but these are public by default!
    int a;
    int b;
    int c;

// but it can have private stuff too
private:
    void f();
}
```

As with namespaces, it is customary not to indent a class block.

By default, members of a class are *private*.
The **only** difference between a `struct` and a `class` is that all members of a `struct` are *public* by default. We will almost always use a `class` unless we essentially just want a data class.

There is a notion of `this` (i.e. `self` in Python), which in a class-defined function call is always a pointer to the class object which called it.
However we prefer to just suffix private/internal members with an underscore.

## Class scope

It's common to declare a class with its method signatures in a header file, then implement those methods in another file.
However, the implementation must scope the class when writing those implementations:

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

## Constructors

Constructors may specify an *initialiser list*, which gives values to data members in order of their member declaration in the class itself:

```cpp
class MyClass {
public:
    MyClass(int i, std::vector<int> j) : i_{i}, j_{j} {
        // ...
    }
private:
    int i_;
    std::vector<int> j_;
}
```

Crucially, this happens *before* the constructor body is actually executed.

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

In other words, initialisation happens for all data members before the body is called, making so-called uniform initialisation more efficient than setting things in the constructor body (since those values are initialised first anyway, perhaps just to default values).
You may as well use an initialiser list to just make those initial values meaningful.

## Delegating constructors

Constructors can call other constructors, possibly to set default values:

```cpp
class MyClass {
public:
    MyClass(int i, std::vector<int> j) : i_{i}, j_{j} {}
    MyClass(std::vector<int> j) : MyClass(6771, j) {};
private:
    int i_;
    std::vector<int> j_;
}
```

## Destructors

Are functions that are called in the moment before an object goes out of scope, which can be useful for cleaning up used resources (e.g. any pointers, opened files or locks owned by the object).
They should not throw exceptions.

```cpp
class MyClass {
    ~MyClass() noexcept;
}

MyClass::~MyClass() noexcept {
    // do destruction, e.g. closing a file
}
```

## Explicit initialisation

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

Sometimes you want this, other times you don't, because implicit type conversions are generally not liked.
To prevent this and force people to do the explicit way, use the `explicit` keyword:

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

// implicit construction is now an error
// Age a = 12;
```

## `const` objects and member functions

By default, member functions are only callable by non-`const` objects.
Only member functions marked with `const` at the end of the function may be called on `const` objects:

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

## Static data members and member functions

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

## Special member functions, `default` and `delete`

By default, the compiler will *synthesise* or create some *special member functions* for you.
Two examples are

- If no constructors are given, then a default no-arg constructor will be created for you
- A copy constructor that allows one to construct an object as a copy of an existing one

To signal to the compiler that you do want its synthesised constructors (e.g. the default constructor), use the keyword `default`.
To signal to the compiler that you *don't* want one of its synthesised constructors (e.g. the copy constructor), use the `delete` keyword.

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

In C++, all operators (like `<`, `==`, `[]`) are functions and can be overloaded to work with custom classes.
Each operator is prefixed by the word `operator` (e.g. the `==` operator is `operator==` as a function).
In general, however, it only makes sense to create an overload for an operator of some type if, when used with that type, it has a single, obvious meaning.

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
