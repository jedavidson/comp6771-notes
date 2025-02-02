# Resource Management

In this section, we discuss the modern principles of resource management in C++.

<!--
## Long lifetimes

There are three ways to make an object's lifetime outlive that of its defining scope:

- Returning out from a function via copy
- Returning out from a function via reference
    - The object itself must always outlive the reference, so references to local function variables can result in undefined behaviour!
- Returning out from a function as a heap resource
-->

## Heap allocation via `new` and `delete`

In C++, the equivalents of `malloc` and `free` are `new` and `delete`.
These call the constructors/destructors of a class to create/delete instances of that class.

```cpp
int* i = new int{6771};
delete i;

std::vector<int>* v = new std::vector<int>{1, 2, 3};
delete v;

int *l = new int[123];
delete[] l; // calls destructor on each elem first before deallocing the array
```

Since the heap is global unlike local function stacks, this means they outlive their defining scopes.

## Destructors

When a non-reference object goes out of scope, the destructor of that object is called, which frees any underlying heap resources that may be under its control.
Examples of where this might be useful:

- Freeing pointers
- Closing open files
- Releasing locks

The process during which these destructor calls are inserted at the end of a scope is called *stack unwinding*.

## Rule of 5/0

When thinking about resource management for a class, there are 5 operations to keep in mind:

- Destructor
- Copy constructor and copy assignment
- Move constructor and move assignment

The *rule of 5* states that if a class defines custom implementations any of the 5 listed operations, then it should also provide custom implementations of the other 4.
The reasoning behind this is that if the default behaviour isn't sufficient for one of them, then it likely isn't sufficient for the others too.
(Prior to C++11, this was the rule of 3, since copy/move assignment didn't exist then.)

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
    char* p;
}

class person {
    // rule of 0: none of the 5 operations are declared and are implicitly defaulted
    // (this is now effectively just a dataclass, i.e. one which holds data without behaviour)
    cstring name;
    int age;
}
```

## Custom copy constructors

Because the default behaviour of a copy constructor is to perform member-wise shallow copies of data members, any pointer resources (e.g. heap arrays) will refer to the same object in both object copies (i.e. changes in one object to this data member reflect in the other).
Even worse, during destruction, such resources will be double-freed.
So, when writing a copy constructor, make sure to do *deep* copies:

```cpp
my_vec::my_vec(mv_vec const& mv)
    : data_{new int[mv.size_]}
    , size_{mv.size_}
{
    std::copy(mv.data_, mv.data_ + mv.size_, data_);
}
```

## Custom copy assignment operators

The *copy-and-swap* idiom is most useful for implementing copy assignment correctly:

```cpp
my_vec::operator=(my_vec const& mv) {
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

## Value categories: lvalues and rvalues

An *lvalue* is an expression that is an object reference, which is to say that it refers to an object with a defined address in memory. On the other hand, an *rvalue* is anything that isn't an lvalue. Things which are lvalues include variable names, and things that are rvalues are non-string object literals (e.g. integers) and return values of functions.

```c++
int a = 3;     //  a = lvalue, 3 = rvalue
int b = f(a);  //  b = lvalue, f(a) = rvalue
int c = b;     //  a, b = lvalue
int d = a + 1; // d = lvalue, a + 1 = rvalue
```

These are the two main *value categories*, [but in reality, it is not a strict dichotomy](https://en.cppreference.com/w/cpp/language/value_category).

Loosely speaking, `std::move` can be used to turn an lvalue into an rvalue. (This is a bit of a lie: in reality, what is created is instead an *xvalue*, or expiring value.)

## Move constructor

Rather than creating new instances of a class by copying, we could instead construct it from the internals of another object by moving:

```c++
// std::exchange(obj, val) replaces the contents of obj by val, and returns the old value
my_vec::my_vec(my_vec&& orig) noexcept
    : data_{std::exchange(orig.data_, nullptr)}
    , size_{std::exchange(orig.size_, 0)}
    , capacity_{std::exchange(orig.capacity_, 0)}
{}
```

These should always be marked `noexcept`, as throwing move constructors are very rare, and by specifying a function as `noexcept`, the compiler can perform many optimisations to improve performance (namely, it doesn't have to worry about generating exception code).

In effect, a new object is created by "stealing" the resources of the moved object.
Afterwards, the moved object should be left in a *valid but indeterminate/unspecified state*.
Validity is dependent on the exact type being worked with (i.e. what constitutes a valid object of one type depends on the requirements for values of that type), however being in an unspecified state means that you cannot determine what its internal state may look like (i.e. you might not be able to say for sure what a specified member field's value will be).

It is bad practice to use a moved-from object after move construction/assignment, and often constitutes undefined behaviour.
A well-configured compiler will warn about this!

## RAII

*Resource Acquisition Is Initialisation* is a C++ programming technique in which resources (i.e. heap objects) are encapsulated inside objects, therefore binding the life cycle of an acquired resource to the lifetime of that object.
We *acquire* the resource in the constructor, and we *release* the resources in the destructor.

Every resource should be owned by one of the following:

- Another resource (e.g. smart pointer, data member)
- A named rsource on the stack
- A nameless temporary variable

## Smart pointers

In C++11 and onwards, smart pointers are a way of wrapping unnamed heap objects (i.e. raw pointers) in named stack objects so that the lifetime of that heap object can be more easily managed (in keeping with the spirit of RAII).

### Unique pointers (`std::unique_ptr`)

A unique pointer is an abstraction that takes *unique* ownership of a heap resource.
When constructed with an initial heap object to manage, the unique pointer is the sole object who owns the managed resource, with all other access to it coming from raw pointers acting as observers.
This forms the common usage pattern with unique pointers: dominion over the resource is established by the unique pointer, which is then held by some object, and any additional references to the underlying heap value are via raw observer pointers.

```cpp
// my_up now owns the heap resources associated with the string "hello"
auto up = std::make_unique<std::string>("hello");

// we use make_unique when constructing them, as it's better than the alternative,
// i.e. explicit use of new (prone to issues regarding use of unnamed temporaries)
auto up = std::unique_ptr<std::string>(new std::string("hello"));

// we can get a raw pointer to the underlying value, a so-called observer
// p_raw will be of type std::string*
auto p = up.get();

// we can access the actual value pointed to using operator* and operator->
auto v = *up;

// we can also relinquish ownership of the resource, and other stuff too
// r will now be a std::string*
auto r = up.release();
```

Since they are unique, these types of pointers are not copy-constructible/assignable (i.e. the copy constructor and copy assignment operator are explicitly `delete`d in their implementations).

Once a unique pointer goes out of scope, its destructor is called.
Naturally, when this destructor is called, its managed heap resource is destroyed.
A consequence is that any observer pointers to a unique pointer that goes out of scope will thus point to garbage memory, so some care has to be taken when using an observer pointer.

### Shared pointers (`std::shared_ptr`)

A shared pointer is an abstraction that allows ownership of an object to be shared across multiple pointers, instead of uniquely owned by one pointer.
This essentially acts as a reference-counted pointer, in that the underlying heap resource is only ever destroyed once the number of shared pointers which point to it hits 0, allowing some pointers to come and go without leading to the demise of that heap object.
This reference count is updated atomically to allow use in concurrent situations.
(There is no non-atomic built-in alternative, unlike e.g. Rust's `Rc` and `Arc`!)

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

Because not all accesses of a value necessarily need to be responsible for the object itself, we can create an analogue of "observers" by using weak pointers.
These do not contribute to the reference count of that heap resource, but if usage of the resource is required (perhaps only temporarily), it must be converted to a shared pointer.
Otherwise, all one can do with a unique pointer is check whether the shared pointer's managed resource is there or not.

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

You should use shared pointers if, for example:

- An object has multiple owners, and it's unclear as to which one will be the longest-lived (e.g. a multithreaded situation needing access to a common resource, where it may not be known which thread exits last)
- You need safe temporary access to an object, which is not guaranteed for observing raw pointers of a unique pointer

These situations are rare in simple use cases, though.

### Smart pointers and partial construction

If an exception is thrown in a constructor, then only some of its subobjects (i.e. fields) are fully constructed.
The C++ standard states that only destructors for these fully constructed subobjects are called, but crucially the constructor of the object itself is not called.

When working with raw pointers, this is a problem, because the destructor of a pointer does nothing, leading to memory leaks.
By using smart pointers, the leak is avoided, as upon subobject destruction, the destructor of a smart pointer is called, which frees underlying memory (at least in the case of a unique pointer).

This implies that, as a general rule of thumb, it is better to manage several wrappers around individual objects which each are responsible/have ownership over that singular resource.
