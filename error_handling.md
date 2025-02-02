# Error handling

In this section, we give an overview of the common ways of handling errors in C++.

## Exceptions and exception objects

Exceptions are a runtime mechanism for signifying exceptional circumstances during code exectution.
Exception handling is the process of managing exceptions that are raised rather than causing the program to crash.

All C++ exceptions are objects which derive from (i.e. are subclasses of) `std::exception`.
As such, we

- throw exceptions by value
- catch exceptions by `const` reference

## Exception control flow

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

## Rethrow

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

## Exception safety

There are four levels of exception safety in C++.

### No-throw exception safety (failure transparency)

An operation that is guaranteed to never throw an unhandled exception provides no-throw exception safety.
While exceptions may occur, they are handled internally. Some examples of such operations:

- Closing files
- Freeing memory
- Move constructors and move assignments
- Trivial stack object creation

### Strong exception safety (commit or rollback)

An operation that may fail, but without leaving visible effects (e.g. no modifications to the object's state) provides strong exception safety.
This is the most common type of exception safety offered by C++ functions.

To achieve this, first perform all throwing operations that don't modify internal state before doing irreversible, non-throwing operations.

### Basic exception safety (no-leak guarantee)

An operation that may fail and cause side effects, but

- respects class invariants
- does not leak resources
- corrupt data

on exception provides basic exception safety.
Objects are afterward left in a *valid but unspecified state*, as there is no telling the extent to which the side effects of partial execution have modified things.

### No exception safety

Operations that make no guarantees regarding exceptions provide no exception safety.
This is often bad C++ code and should be avoided at all costs (especially since wrapping resources and attaching lifetimes to them can give at least basic exception safety).

## The `noexcept` keyword

Functions marked as `noexcept` are understood to not throw unhandled exceptions (but doesn't technically prevent them from doing so).
STL functions can operate more efficiently on `noexcept` functions.

## Type-level exceptions and `std::expected`

[TODO: cover the C++23 type `std::expected`, which gives you Rust-like `Result` type (but worse lol)]
