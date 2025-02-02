# STL I: Containers

In this section, we give an overview of the main containers (i.e. data structures) on offer in the **S**tandard **T**emplate **L**ibrary.

## Sequential containers

These organise a finite set of objects into a strict linear arrangement:

- `std::vector` is a dynamically-sized array, and the most common container to use
    - Initial capacity = #. of initial elements that can be stored
    - When the size of the vector reaches capacity, the capacity is doubled
    - Does not automatically shrink capacity, but one can use `v.shrink_to_fit()` to do that on demand
    - Provides two ways to access items:
        - `v.at(i)` does bounds checking (safe, more expensive)
        - `v[i]` doesn't do bounds checking (lightweight, can lead to undefined behaviour)
- `std::array` is a more lightweight wrapper for a C-style fixed size array
    - Crucially, while `std::vector` places its memory in the heap, the underlying array in an `std::array` lives in the stack, which can save some system calls
    - The sheer flexibility of `std::vector` means you probably want to use that most of the time unless a specific enough situation occurs for `std::array` to be suitable
- `std::deque` is a double-ended queue
    - Its implementation is [a bit more intricate](https://stackoverflow.com/questions/6292332/what-really-is-a-deque-in-stl) than, say, a ring buffer which one might expect to be the underlying design
- `std::forward_list` is a singly-linked list
- `std::list` is a doubly-linked list

Most operations on these containers are either $O(1)$ or amortised $O(1)$.
The performance of the last 3 can be hindered by cache locality concerns.

## Ordered associative containers

These provide fast key-based retrieval of data, but with an order on the elements:

- `std::set` is a set in the mathematical sense
- `std::multiset` is a multiset in the mathematical sense
- `std::map` is a hash table
- `std::multimap` is a hash table with non-unique keys

The ordering here is element sorted order (for `std::map` and `std::multimap`, this is by key), not insertion order. This is achieved by storing them as a search tree (e.g. a red-black tree), which gives most operations on them costs of $O(\log{n})$.

A non-default ordering for these containers can be specified by providing a custom element comparator function when constructing it:

```cpp
// this is an int -> int map, but the keys are sorted in descending order
auto m = std::map<int, int, std::greater<int>>{};
```

## Unordered associative containers

These provide even faster key-based retrieval of data via hashing, at the cost of any guaranteed ordering of the elements:

- `std::unordered_set` is the unordered version of `std::set`
- `std::unordered_multiset` is the unordered version of `std::multiset`
- `std::unordered_map` is the unordered version of `std::map`
- `std::unordered_multimap` is the unordered version of `std::multimap`

The average complexity of most operations on these is $O(1)$.

[TODO: write about the implementation of `unordered_map` being a separately chained hash table.]

## Container adapters

These restrict the functionality of an existing container to provide a different set of functionalities:

- `std::stack` is a LIFO stack
- `std::queue` is a FIFO queue
- `std::priority_queue` is a queue where larger elements leave before smaller elements (i.e. it behaves like a max heap)

When declaring container adapters, the underlying sequence container can be specified.
For example, by default `std::stack` will use a `std::deque` as its underlying representation.

As with the ordered associative containers, a non-default ordering can be specified for a `std::priority_queue`:

```cpp
// this is a priority queue of ints where smaller elements leave before larger elements
// (i.e. it behaves like a min heap)
auto pq = std::priority_queue<int, std::vector<int>, std::greater<int>>{};
```

## Inserters and back inserters

- `std::inserter(c, it)` returns an iterator that allows for the insertion of elements at the location of `it` in the container
- `std::back_inserter(c)` returns an iterator that allows for the insertion of elements at the end of the container

## Asides

For certain data structures, there is an `emplace` and a `push/insert/...` function.
The difference is that `emplace` constructs an object in-place without using copies or moves by some forwarding magic:

```cpp
auto x = std::stack<MyObj>{};
x.push(MyObj(1, 2, 3)); // creates a temporary copy of the object, then moves it into the stack
x.emplace(1, 2, 3);     // forwards the arguments to the MyObj constructor to create it in the stack in-place
```

This can often be a faster way of doing it as you avoid moves/copies.

Some types cannot be stored in containers:

- References
    ```cpp
    // this is a compile error
    // the full reasoning for this is rather technical, but the crux of the issue
    // is that per the C++ specification, pointers to references are illegal
    // https://eel.is/c++draft/container.requirements#container.reqmts-note-2
    auto vec = std::vector<int&>{};

    // plain pointers can be stored instead, as can a std::reference_wrapper
    // (the latter is a reference replacement that is, among other things, assignable)
    auto vec1 = std::vector<int*>{};
    auto vec2 = std::vector<std::reference_wrapper<int>>{};
    ```
- `const` objects
    ```cpp
    // also a compile error, but again the reasoning for this is slightly technical
    // pre-C++11, the problem was that containers required the inner type to be assignable
    // (which const objects aren't), but this is not true anymore
    // nevertheless, compilers will reject this
    auto vec = std::vector<int const>{};
    ```
