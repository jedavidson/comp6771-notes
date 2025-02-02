# Dynamic Polymorphism

In this section, we cover polymorphism (as it is meant in the typical OOP sense) in C++.

Polymorphism is the provision of a single interface to entities of different types.
When this occurs at runtime, we call it dynamic polymorphism. In contrast to some other language, this is (at least for the most part) without performance penalty, in keeping with the C++ ethos of "you don't pay for what you don't use".

## Inheritance

```cpp
class base {
public:
    int x;
    auto foo() -> void;

// as in other languages, protected members are accessible only to objects in a subclass hierarchy
// (i.e. base itself, and any class which derives from base)
protected:
    int y;
    auto bar() -> void;

private:
    int z;
    auto baz() -> void;
};

// note the access specifier!
class derived : public base {
public:
    // we can 'override' functions like usual
    auto foo() -> void;
};
```

The access specifier above dictates the maximum level of accessibility of things inherited from the base class:

```cpp
// public inheritance essentially changes nothing about accessibility
class derived : public base {
    // ...
};

// with protected inheritance, the inherited public members of base have their accessibility
// capped at protected, i.e. no longer part of derived's public interface
class derived : protected base {
    // ...
};

// with private inheritance, the inherited public and protected members have their accessibility
// capped at private, i.e. only accessible from within derived itself
class derived : private base {
    // ...
};

// the access specifier is optional, and if it is left out, it's just private inheritance
class derived : base {
    // ...
};
```

Unless you have a good reason, we typically do public inheritance.

The member variable layout in memory of a derived class is that of contiguous subobjects:

```
|^^^^^^^^^^^^^^^^^^^^^^^^^^^^^|
| base subobject:             |
| - member variables          |
|-----------------------------|
| derived subobject:          |
| - non-base member variables |
|_____________________________|
```

## Object slicing problem

Since the objects of a derived class may be larger than objects of its base class, it's unclear to the compiler how much space one would need to allocate in a situation like this:

```cpp
// since a class could derive from base, how much space do we allocate on the stack
// to hold the obj argument?
auto do_something(base obj) {
    obj.say_hi();
}
```

The only sensible answer is to allocate enough size for a base class object.
However, a consequence of this is that when a derived class object is passed by value as an argument to `do_something`, the copy of the object residing in the memory allocated for the `obj` argument only contains the data of the base class subobject.
This is the *object slicing problem*: the additional memory usually contained in a derived class object has been "sliced off" in the copy.

Passing by reference (i.e. references or raw pointers) will avoid object slicing, so we always prefer this when dealing with inheritance hierarchies.

## Dynamic binding

When passing base class objects by reference, C++ uses, by default, the base class implementation of functions, even if they have been overriden in a derived class:

```cpp
class base {
public:
    auto say_hi() -> void {
        std::cout << "Hi from the base class\n";
    }

    auto say_bye() -> void {
        std::cout << "Bye!\n";
    }
};

class derived : public base {
public:
    auto say_hi() -> void {
        std::cout << "Hi from the derived class\n";
    }
};

auto do_something(base& obj) {
    obj.say_hi();
}

do_something(base{});    // prints "Hi from the base class" (fine)
do_something(derived{}); // prints "Hi from the base class" (weird)
```

This is mostly a performance consideration; this default is easily determined at compile time, since derived class object references can be *statically bound* to base class object references.

However, C++ can be forced to use *dynamic binding* to determine the right override to pick at runtime using *virtual functions*:

```cpp
class base {
public:
    // the virtual keyword indicates that this function may be overriden in a derived class,
    // and so C++ must now put in the effort to determine which one to call from the context
    virtual auto say_hi() -> void {
        std::cout << "Hi from the base class\n";
    }

    virtual auto say_what() -> void {
        std::cout << "What?\n";
    }

    // this function is non-virtual, i.e. normal
    auto say_bye() -> void {
        std::cout << "Bye!\n";
    }
};

class derived : public base {
public:
    // the override keyword indicates that it is an override of a base class function
    auto say_hi() -> void override {
        std::cout << "Hi from the derived class\n";
    }

    // the subclass doesn't have to override *every* virtual from the base class
};

auto do_something(base& obj) {
    obj.say_hi();
}

do_something(base{});    // prints "Hi from the base class"
do_something(derived{}); // prints "Hi from the derived class"
```

The `override` keyword is technically optional, but there are benefits to using it:

- If there is no virtual base class function of the same name, using `override` will pick this up at compile-time
- It's less ambiguous and error-prone for programmers

Virtual functions rely an under-the-hood abstraction called a *vtable*.
Each class has its own vtable, stored in the data segment, consisting of an array of function pointers to the definition of each virtual function.

```
in the code segment, we have
    base::say_hi() { /* ... */ }
    derived::say_hi() { /* ... */ }
    base::say_what() { /* ... */ }

in the data segment, we have
    base_vtable:    [<pointer to base::say_hi>,    <pointer to base::say_what>]
    derived_vtable: [<pointer to derived::say_hi>, <pointer to base::say_what>]
```

If a class has a non-empty vtable, each object of that class internally also holds a pointer to the vtable of its class to aid in dynamic binding.
When a virtual function is called on such an object passed by reference (i.e. either by reference or pointer type), C++ will

- Follow the object's vtable pointer
- Use offset arithmetic (specific to each virtual function) to find the correct function pointer in the vtable
- Follow this function pointer to get to the definition of the virtual function and call it

This involves additional runtime overhead beyond a normal function call (more machine instructions are required), and also incurs indirect memory accesses (which can have poor cache performance), so is undesirable in performance-sensitive code. Sometimes, compilers can [devirtualise](https://quuxplusone.github.io/blog/2021/02/15/devirtualization/) virtual function calls to avoid this extra work.

[TODO: default args and virtuals]

## Finality

We can specify that a virtual function in a derived class will not be further overriden by any of its own derived classes, i.e. will not be virtual further down the inheritance hierarchy:

```cpp
class base {
public:
    virtual auto say_hi() -> void {
        // ...
    }
};

class derived : public base {
public:
    // the final keyword indicates that no class which inherits from derived will
    // also override this function, and a compile error will be generated if
    // one attempts to do so
    auto say_hi() -> void final override {
        // ...
    }
};

// even though obj could refer to objects of a class which inherit from derived,
// they won't override say_hi by finality, so we can just do static binding here rather
// than dynamic binding, which eliminates the performance overhead of the vtable
auto do_something(derived& obj) {
    obj.say_hi();
}
```

Finality can also be applied to classes themselves, preventing other classes deriving from it:

```cpp
class base final {
    // ...
};

// this produces an error at compile time now
class derived : public base {
    // ...
};
```

## Pure virtual functions and abstract classes

A virtual function is considered to be *pure virtual* if it has no corresponding implementation in that class:

```cpp
class base {
public:
    // the pure specifier "= 0" indicates that this base class virtual function
    // doesn't come with an implementation of say_hi,
    // so in effect this just acts as an interface
    virtual auto say_hi() -> void = 0;

    virtual auto say_bye() -> void {
        std::cout << "Bye!\n";
    };
};

class derived : public base {
public:
    auto say_hi() -> void override {
        // we now have to implement it in any derived classes
        // say_hi becomes non-pure virtual for any further subclasses of derived
    }
};

class intermediate : public base {
public:
    // we can also have pure virtual overrides of non-pure virtual functions
    // anything which derives from intermediate must implement say_bye
    auto say_bye() -> void override = 0;
};
```

If a class has at least one pure virtual member function, then objects of that class *cannot* be constructed.
Moreover, functions cannot accept or return objects of such a class type by value, only by reference.

Pure virtual functions allow us to mimic the behaviour of *abstract classes* from other OOP languages.

## OOP type theory

[TODO: covariance, contravariance, ...]

## Polymorphism and construction

To avoid the object slicing problem, we must use pointers to store polymorphic objects in, say, a container:

```cpp
// this doesn't work, because all of the contents of the vector are stored inline,
// which introduces object slicing
auto objs = std::vector<base>{};
objs.push_back(base{});
objs.push_back(derived{});

// we know we can't store references, so we must do pointers (raw or smart)
auto objs = std::vector<std::unique_ptr<base>>{};
objs.push_back(std::make_unique<base>());
objs.push_back(std::make_unique<derived>());

// TODO: there is, supposedly, a subtle problem with this code,
// but i don't see it right now
```

Since derived class objects contain base class subobjects, derived classes must call a base class constructor:

```cpp
class base {
public:
    base(int x) : x_{x} {}
private:
    int x_;
};

class derived : public base {
public:
    // derived constructor calls base constructor
    // if it doesn't, then the default constructor of base is implicitly called
    derived(int x, int y) : base(x), y_{y} {}
private:
    int y_;
};
```

Initialisation of the base subobject within a derived class object cannot be done within the derived class, even for protected members.

[TODO: finish this]

## Polymorphism and destruction

[TODO]
