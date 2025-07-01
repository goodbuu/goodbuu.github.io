---
title: C++ Virtual Functions Stack Up Debuffs
published: true
---

The call overhead of virtual functions is well known, but it is often one of the smaller costs. Other costs can be spread throughout the program and difficult to spot in a profiler.

All code in this post is compiled with Clang without `-fstrict-vtable-pointers` (off by default).

## Design-level costs

Dynamic binding of virtual functions happens when an object of a derived class is accessed through a pointer or a reference to the base class. The typical approach is to create objects dynamically and store pointers to the base class.

```cpp
class Base {
 public:
  virtual ~Base() = default;
  virtual void Foo() = 0;
};

class Derived : public Base {
 public:
  void Foo() override {}
};

constexpr int kN = 1000;
std::vector<Base*> v;
v.reserve(kN);
for (int i = 0; i < kN; ++i) {
  v.push_back(new Derived);
}
for (Base* ptr : v) {
  ptr->Foo();
}
for (Base* ptr : v) {
  delete ptr;
}
```

Compared with storing the objects in place, creating them dynamically requires 1,000 more memory allocations and deallocations. A memory allocation can be very expensive on the slow path, where the allocator may have to request more memory from the operating system.

## Compiler-level costs

A polymorphic object contains a vptr, which points to the vtable of its class. The vtable holds pointers to the virtual functions. A virtual call is therefore an indirect call, and its overhead is more significant for small functions.

```asm
movq  (%rdi),%rax
callq *16(%rax)
```

For `ptr->Foo()`, `ptr` is in `rdi`. The vptr of `*ptr` is loaded into `rax`, and `Foo` is called through the vtable entry at offset 16.

Because the call target is not known at compile time, the compiler cannot inline a virtual function, so every call pays the overhead. Worse, the compiler loses the optimization opportunities that inlining would expose.

```cpp
class Base {
 public:
  virtual ~Base() = default;
  virtual int Foo(int x) = 0;
};

class Derived : public Base {
 public:
  int Foo(int x) override { return x * (a_ + b_); }
 private:
  int a_ = 1;
  int b_ = 2;
};

auto dist = std::bind(std::uniform_int_distribution<>(0, 99),
                      std::default_random_engine());
std::vector<int> arr(1000);
std::generate(arr.begin(), arr.end(), dist);

Base* ptr = new Derived;
int sum = 0;
for (int x : arr) {
  sum += ptr->Foo(x);
}
delete ptr;
```

Inlining `Foo` would enable two loop optimizations. Loop-invariant code motion hoists `a_ + b_` out of the loop, because `a_` and `b_` do not change across iterations. Vectorization turns the loop into SIMD instructions that process multiple elements of `arr` at once.

## Hardware-level costs

A container such as `std::vector<Base*>` stores pointers contiguously, but the objects they point to are allocated separately. Because the objects may be scattered across memory, loading their vptrs and reading or writing their data members can hurt cache locality.

When the objects in a container belong to multiple derived classes and are stored in an irregular order, consecutive calls dispatch to different overrides of the same virtual function. These overrides, especially large ones, can evict one another from the instruction cache.

To the CPU, a virtual call is an indirect branch, and the call target is predicted from the branch history. When the call target changes irregularly, the CPU often mispredicts and has to flush the pipeline.

## Conclusion

- Use `boost::base_collection` to store polymorphic objects in place and group them by derived class.
- Use the curiously recurring template pattern (CRTP) when the derived class is known at compile time.
