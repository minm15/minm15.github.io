# STL

The C++ Standard Library provides containers, iterators, algorithms, and utilities that cover most application-level data structure needs.

## Containers

- `std::vector` is usually the default sequential container.
- `std::deque` is useful when growth at both ends matters.
- `std::map` and `std::set` provide ordered tree-based lookups.
- `std::unordered_map` and `std::unordered_set` provide hash-based average O(1) lookup.

## Algorithm Mindset

Prefer standard algorithms over handwritten loops when the intent is clearer:

```cpp
std::sort(values.begin(), values.end());
auto it = std::find(values.begin(), values.end(), target);
```

## Common Pitfalls

- Iterator invalidation rules differ by container.
- Associative containers trade memory and pointer chasing for stable ordering.
- Hash containers need good hash behavior to keep lookup costs predictable.
