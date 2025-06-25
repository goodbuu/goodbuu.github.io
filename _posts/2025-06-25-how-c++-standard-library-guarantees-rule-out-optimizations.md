---
title: How C++ Standard Library Guarantees Rule Out Optimizations
published: true
---

The C++ standard library makes strong guarantees about its behavior. These guarantees help users write reliable generic code, but they also rule out some optimizations.

## Small-buffer optimization

Small-buffer optimization (SBO) uses a fixed-size inline buffer embedded in the object to store small amounts of data. When the data fits in this buffer, no separate dynamic allocation is needed.

```cpp
std::vector<int> a{1}, b{2, 3};
auto it = a.begin();
int* ptr = &a[0];
int& ref = a[0];

a.swap(b);

assert(it == b.begin());
assert(ptr == &b[0]);
assert(&ref == &b[0]);
```

The C++ standard requires `std::vector::swap` to exchange the contents of two vectors without moving, copying, or swapping individual elements. It also requires iterators (except `end()`), pointers, and references to remain valid after the swap. They still refer to the same elements, which now belong to the other vector.

These guarantees prevent `std::vector` from using SBO. If the elements were stored in inline buffers, `swap` would have to move or swap individual elements and could not ensure that existing iterators, pointers, and references continued to refer to the same elements in the other vector.

In contrast, `std::string::swap` is permitted to invalidate iterators, pointers, and references, so implementations can use small-string optimization (SSO).

Vector-like containers that use SBO include `boost::container::small_vector`, `absl::InlinedVector`, and `llvm::SmallVector`. These containers fall back to dynamic allocation when the data does not fit in their inline buffers.

## Flat hash tables

The discussion below focuses on maps, but the same points apply to sets. `std::unordered_map` typically uses separate chaining, in which each element is stored in a separately allocated node and the nodes in the same bucket are linked together. Insertions of new elements need node allocations, which add overhead. Lookups follow pointers to nodes that may be scattered across memory, which can hurt cache locality.

Flat hash tables store elements directly in a slot array and use open addressing to resolve collisions by probing other slots. This avoids separate node allocations and pointer chasing, and can improve cache locality.

However, the C++ standard requires `std::unordered_map` to provide the following guarantees:

- Pointers and references to elements must remain valid after rehashing, whether it is triggered by insertion or by a call to `rehash` or `reserve`. Moving a flat hash table's elements into a new slot array would change their addresses.
- The modifiers `extract`, `merge`, and `insert(node_type&&)` must transfer ownership without copying or moving elements. The elements therefore need to be stored in separately allocated nodes that can be detached from the container, rather than directly in a slot array.
- The bucket interface exposes how elements are grouped into buckets and allows iteration within each bucket. Separate chaining supports this interface with the required semantics and complexity more naturally than open addressing.

Flat hash table implementations include `boost::unordered_flat_map`, `absl::flat_hash_map`, and `llvm::DenseMap`. These tables often provide faster insertions and lookups, especially when keys and values are small.
