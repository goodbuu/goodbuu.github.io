---
title: When the Meyers Singleton Is Not Free
published: true
---

The Meyers singleton is the usual way to implement the singleton pattern in C++. It is simple to write, and since C++11 its initialization is guaranteed to be thread-safe. However, that simplicity is not always free.

## The cost of the guard variable

```cpp
class LocalStatic {
 public:
  static LocalStatic& GetInst() {
    static LocalStatic inst;
    return inst;
  }
  std::vector<int> v;
 private:
  LocalStatic() = default;
  ~LocalStatic() = default;
};
```

`GetInst` returns a reference to `inst`, a function-local static object. The first call to `GetInst` performs the one-time work that `inst` needs. If another thread calls the function at the same time, that thread will wait for the work to finish.

Compilers typically use a guard variable for that work. The first call sets the guard variable once the work is finished, and later calls check it before returning `inst`. That check stays on the hot path for the whole run of the program.

## Managing the object lifetime by hand

One way to remove that check is to manage the object lifetime yourself.

```cpp
class ManualLifetime {
 public:
  static void ConstructInst();
  static void DestructInst();
  static ManualLifetime& GetInst();
  std::vector<int> v;
 private:
  ManualLifetime() = default;
  ~ManualLifetime() = default;
};

alignas(ManualLifetime) std::byte InstMem[sizeof(ManualLifetime)];

void ManualLifetime::ConstructInst() { ::new (InstMem) ManualLifetime(); }
void ManualLifetime::DestructInst() { GetInst().~ManualLifetime(); }
ManualLifetime& ManualLifetime::GetInst() {
  return *std::launder(reinterpret_cast<ManualLifetime*>(InstMem));
}
```

`InstMem` is an array of `std::byte` with the same size and alignment as `ManualLifetime`. It has static storage duration, so it exists for the whole run of the program.

`ConstructInst` uses placement new to construct the object in `InstMem`, which starts the object's lifetime without allocating memory. `DestructInst` calls the destructor directly, which ends the object's lifetime without releasing memory. You must call `ConstructInst` and `DestructInst` exactly once each, and `GetInst` only between them.

`GetInst` returns a reference to the object in `InstMem` without checking a guard variable. Casting `InstMem` to `ManualLifetime*` gives a pointer to the storage, not to the object whose lifetime started there. `std::launder` is therefore needed to obtain a pointer to the object.

## Comparing the two versions

Benchmark: [Quick C++ Benchmarks](https://quick-bench.com/q/yQmXUUxYaT3CljmRJronnZHI0LA)

Both versions call `GetInst` and read `v.size()`.

`ManualLifetime` is about 1.7x as fast as `LocalStatic`.

The hot path of `LocalStatic` has three more instructions, which load the guard variable, test it, and jump to the cold path when it is not set. They run on every call to `GetInst`.

```asm
movzbl guard_variable(%rip),%eax
test   %al,%al
je     cold_path
```

## When the compiler emits a guard variable

Benchmark: [Quick C++ Benchmarks](https://quick-bench.com/q/jj4q0biF4BV4FMNHf1Iz9tMJjw0)

In both classes, the `std::vector<int>` member is replaced with an `int`. Everything else is the same.

Both versions perform similarly.

The first call to `GetInst` has work to do in two cases:

- `inst` cannot be initialized at compile time, so it needs dynamic initialization.
- `inst` has a non-trivial destructor, which has to be registered to run at exit.

A `std::vector<int>` member has a non-trivial destructor, which is why the first benchmark has a guard variable. With an `int` member, neither case applies. `inst` is initialized at compile time and its destructor is trivial, so the first call has no work to do and there is no guard variable. After `GetInst` is inlined, each version compiles to a single load from static storage.

## Conclusion

- If performance-critical code accesses a Meyers singleton, check whether the compiler emits a guard variable and measure what it costs.
