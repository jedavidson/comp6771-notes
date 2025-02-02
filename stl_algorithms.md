# STL III: Algorithms

In this section, we discuss the `<algorithm>` header, which gives some nice standardised functions for common tasks to cut down on boilerplate code that the programmer has to write.
These algorithms work on iterators to containers rather than containers directly so as to be most portable with different containers.

## Map and reduce equivalents

```cpp
// this is map, which places the mapped contents into a destination container
// it is up to the programmer to make sure that destination container is big enough!
std::transform(src_begin, src_end, dest_begin, func);

// this is basically a non-returning version of map
// the func should be void, because its return type is ignored
// to mutate, make sure func takes references in the argument, and change via assignment
std::for_each(begin, end, func);

// this is reduce
// function takes (accumulator, value) as arguments (in that order)
auto result = std::accumulate(begin, end, initial_value, func);
```

For more functional-esque tools, see the `<functional>` header (similar to Python's `operator` and `functools` modules).

[TODO: expand a bit to cover some others and some examples of using, say, lambdas with them]