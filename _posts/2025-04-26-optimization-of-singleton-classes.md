---
published: true
---

Given the widespread use of the singleton class in practice, it's important to comprehend its disadvantages.

# Meyers's singleton
```cpp
class Singleton {
 public:
  static Singleton& GetInst() {
    static Singleton inst;
    return inst;
  }
  int i;
 private:
  Singleton() = default;
  ~Singleton() = default;
};
```

This popular implementation uses the basic properties of static local variables.
It offers the benefits of lazy initialization and thread safety (after C++11).

The destruction order of static local variables in different translation units is undefined.

Most compilers guarantee thread safety for static local variables through DCL (also known as double-checked locking).
However, DCL requires some additional MOV and TEST instructions to check whether static local variables have been initialized, which will cause performance loss.

# How to optimize
```cpp
class SingletonV2 {
 public:
  static void ConstructInst();
  static void DestructInst();
  static SingletonV2& GetInst();
  int i;
 private:
  SingletonV2() = default;
  ~SingletonV2() = default;
};

char InstMem[sizeof(SingletonV2)];

void SingletonV2::ConstructInst() { new (InstMem) SingletonV2; }
void SingletonV2::DestructInst() { GetInst().~SingletonV2(); }
SingletonV2& SingletonV2::GetInst() {
  return reinterpret_cast<SingletonV2&>(InstMem);
}
```

Usually, a clear destruction order is more important than lazy initialization.
Explicitly initializing once at the beginning of the program can avoid race conditions.

When the operating system loads a binary or library, it allocates the memory for global variables without requiring release.
Using placement new can call the constructor of the singleton class on this memory.

# Measure them
The benchmark is here: [quick-bench](https://quick-bench.com/q/fUx924fJLCHMMFhUpFouWTd5CDk)

Why do the two versions have the same performance?
The answer is the complexity of the singleton class.
The only data member is int, so the two versions compile to almost the same assembly.

The benchmark for replacing int with vector&lt;int&gt; is here: [quick-bench](https://quick-bench.com/q/znUd-c_JA2MqAyGioEbBDhe2www)

V2 is 1.7X faster.
Here are additional instructions:
```nasm
movzbl 0x3b869(%rip),%eax
test   %al,%al
```

# Conclusions
- Be careful with static local variables on a hot path.
- Measure your code, change it, and measure it again.
