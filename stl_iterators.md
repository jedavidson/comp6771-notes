# STL II: Iterators

In this section, we introduce *iterators*, the abstract interface for moving through the items in an STL container.

## Creating iterators

Many different flavours:

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

## Interacting with iterators

Iterators behave a lot like pointers and not references, so you need to use `*` to actually get the item that the iterator is pointing to at any particular time.
Dereferencing an `end` iterator is undefined behaviour.

There are two functions for advancing iterators non-linearly (i.e. besides doing `++`):

- `std::advance(it, n)` will advance the iterator `n` steps
- `std::next(it, n)` will create a copy of the iterator advanced `n` steps

In situations where the length of the underlying container may not be known/stored, there is a way to calculate the "distance" between two iterators:

```cpp
auto dist = std::distance(it1, it2);
```

While it makes it a bit more verbose, this approach is useful in certain situations where you may not have a *random-access iterator*.
For example, taking the midpoint of two iterators:

```cpp
// only works if the iterator given back is random-access
auto mid_it_ra = (container.begin() + container.end()) / 2;

// works regardless, though it is a bit longer
auto mid_it = std::next(container.begin(), std::distance(container.begin(), container.end()) / 2);
```

(This, however, is *not* a constant time operation if the iterator is not random-access!)

## Types of iterators

- Input iterator: read-only iterator that can be incremented or compared for (in)equality
- Output iterator: write-only iterator that can be incremented or compared for (in)equality
- Forward iterator: like an input/output iterator but you can both read and write
- Bidirectional iterator: like a forward iterator but you can decrement too
- Random-access iterator: most general kind of iterator, which, in addition to all of the previously listed features, provides
    - Relational comparisons (e.g. `it1 < it2`)
    - Iterator arithmetic (e.g. `it1 + it2`)
    - Subscript access to values (e.g. `it[k]`, which is just `*(it + k)`)

In general read/write situations, we have forward iterators $\subset$ bidirectional iterators $\subset$ random-access iterators.

Different STL containers provide different iterator types:

- `std::vector`, `std::deque` and `std::array` give random-access iterators
- `std::list`, `std::(multi)set` and `std::(multi)map` give bidirectional iterators
- `std::forward_list`, `std::unordered_(multi)set` and `std::unordered_(multi)map` give forward iterators

Container adapters do not provide iterators.

In C++20, there are also contiguous iterators, which are random iterators that guarantee contiguity of the underlying elements in memory.

## Iterator invalidation

When we modify a container, this may affect iterators to that container.
For example, if we are continually moving the endpoint of a container by inserting/deleting elements from it, it is likely not the case that an old `end()` iterator is valid anymore.
This is *iterator invalidation*: some operations may render existing iterators invalid, such that continued use of these iterators is now undefined behaviour.

Which operations do and don't invalidate iterators should be specified by the operations of the container.
Some operations on some containers may only invalid some iterators (e.g. it might only invalidate the past-the-end `end()` iterator).

## Creating custom iterators

### Iterator traits

Each iterator has certain properties, referred to as the *iterator traits*:

- Category: is it an input/output, forward, bidirectional or random-access iterator?
- Value type: what is the type of the element that the iterator points to?
- Reference type: what is the type of references to elements that the iterator points to?
- Pointer type: what is the type of pointers to elements that the iterator points to?
- Difference type: what is the type that results upon subtraction of iterators?

### Building iterators

An iterator, at minimum, must look like this:

```cpp
#include <iterator>

template <typename T>
class my_iterator {
public:
    using iterator_category = std::forward_iterator_tag; // or some other iterator category
    using value_type = T;
    using reference = T&;
    using pointer = T*;
    using difference_type = int;

    my_iterator& operator++();
    my_iterator operator++(int) {
        auto copy{*this};
        ++(*this);
        return copy;
    }

    reference operator*() const;

    // not strictly required, but it's nice to have
    pointer operator->() const {
        return &(operator*());
    }

    // in C++20, operator!= is automatically derived for you from operator==
    friend bool operator==(const my_iterator& lhs, const my_iterator& rhs) {
        // ...
    }
};
```

For bidirectional iterators, one also needs to provide prefix and postfix `operator--`.

To allow a custom container to be used with STL functions that accept iterators, all one needs to do is provide implementations of `begin()`, `end()`, `cbegin()` and `cend()`. Within the container, one should also specify some iterator types:
```cpp
using iterator = // your iterator type here
using const_iterator = // your const iterator type here
```

If you have a bidirectional iterator already, you can get reverse iterators for free by using the `reverse_iterator` and `const_reverse_iterator` iterator adaptors.

## Asides

When using a `const` iterator in a loop for example, you don't make the actual iterator variable `const` explicitly:

```cpp
for (auto const it = c.begin(); it != c.end(); ++it) {
    // doesn't work, needs to be auto it = ...
}
```

When working with something like a `map`, if you want to look up whether an item is in the map and then use that item if it does exist, it is often quicker to work with the `.find()` method, which will give back an iterator:

```cpp
// iterator method: one lookup and can access the item at that key via the iterator
// note: the iterator is to a key-value std::pair, hence the use of ->
auto it = map.find(key);
if (it != map.end()) {
    auto v = it->second;
}

// since C++17, you can actually put an init statement inside an if, which is nice if
// you have no use for the iterator variable after the body of the if is executed
if (auto it = map.find(key); it != map.end()) {
    auto v = it->second;
}

// since C++20, there is a map.contains() method to check existence,
// but accessing the value afterwards requires a second lookup
if (map.contains(key)) {
    auto v = map.at(key);
}
```

[TODO: STL-compliant iterator operations must be amortised $O(1)$, and the consequences this has for `std::unordered_map`]