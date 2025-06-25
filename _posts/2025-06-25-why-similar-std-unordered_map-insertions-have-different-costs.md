---
title: Why Similar std::unordered_map Insertions Have Different Costs
published: true
---

Background: In `std::unordered_map<K, T>`, `value_type` is an alias for `std::pair<const K, T>`. The `const` qualifies only `K`. The `value_type` object itself can be const or non-const.

## Emplace is not always faster than insert

Benchmark: [Quick C++ Benchmarks](https://quick-bench.com/q/jqZIxpEKEgy-eaSqu2juZedQ1qU)

The benchmark uses libstdc++. Both benchmark functions create a map and repeatedly try to insert the same key. Only the first insertion succeeds. Subsequent insertions fail because the key is already in the map.

`BM_InsertPair` is about 5.6x as fast as `BM_EmplaceArgs`.

The cost difference comes down to when a node is created. Creating a node means allocating memory and constructing a `value_type` object. `BM_EmplaceArgs` hands the key and mapped value to `emplace` as constructor arguments for `value_type`. `emplace` creates the node first, then reads the key from the constructed `value_type` object to do the lookup. If the key is already present, it discards the node. `BM_InsertPair` hands a const `value_type` object to `insert`. `insert` can access the key directly, so it does the lookup first and creates a node only if the key is absent.

libc++ uses a key-extraction optimization to eliminate this extra cost. When the first argument's type is `K` after applying `std::remove_reference_t` and `std::remove_const_t`, `emplace` extracts it to do the lookup. If the key is available on its own, `try_emplace` is the more portable choice: it guarantees that no `value_type` object is constructed if the key is already in the map.

*Update: Starting with version 15, libstdc++ also uses a key-extraction optimization.*

## How const affects insert overload resolution

```cpp
std::unordered_map<std::string, int> m;
const std::pair<const std::string, int> cp;
std::pair<const std::string, int> p;

m.insert(cp); // resolves to insert(const value_type&)
m.insert(p);  // resolves to insert(P&&), where P is value_type&
```

Both `cp` and `p` are lvalues, but only `cp` is const. For each call, two `insert` overloads are viable: the non-template overload `insert(const value_type&)` and the template overload `insert(P&&)`.

For `m.insert(cp)`, `P` is deduced as `const value_type&`, so `P&&` collapses to `const value_type&`. Both overloads therefore have the same parameter type, and neither conversion sequence is better than the other. Given this tie, overload resolution prefers the non-template overload.

For `m.insert(p)`, `P` is deduced as `value_type&`, so `P&&` collapses to `value_type&`. The reference-binding rules prefer the less cv-qualified type, so binding `p` to `value_type&` is a better conversion sequence than binding it to `const value_type&`. Overload resolution therefore selects the template overload.

The standard specifies that `insert(P&& obj)` is equivalent to `emplace(std::forward<P>(obj))`, so a failed insertion may create and discard a node.

## Conclusion

- Prefer `try_emplace` to `emplace` when the key is available on its own.
- When passing an lvalue `value_type` object to `insert`, make it const.
