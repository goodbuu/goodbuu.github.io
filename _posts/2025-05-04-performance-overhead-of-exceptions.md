---
title: Performance Overhead of Exceptions
published: true
---

There are many stories about the performance overhead of exceptions, but what is the truth?

# Measure throw
The benchmark is here: [quick-bench](https://quick-bench.com/q/it0NASOaOz0lQqF7LLD5I7ZwGQg)

HandleErrorWithReturn is 15X faster than HandleErrorWithThrow.
There are three main reasons why exceptions are so slow:

Throwing an exception will cause dynamic memory allocation.

Catching an exception will cause RTTI (also known as runtime type identification).

The thrown exception object requires stack unwinding to find a catch statement that matches the type.

# Measure catch
The benchmark is here: [quick-bench](https://quick-bench.com/q/1Z8G1MgJgFq30NY1g8_JrGHuxlo)

Both versions have the same performance.
However, catching exceptions significantly increases the number of instructions, which may cause L1 cache misses.

# Conclusion
- Do not attempt to change the control flow of your program by throwing exceptions.
- You don't need to be concerned about the performance overhead of exceptions that won’t actually be thrown.
