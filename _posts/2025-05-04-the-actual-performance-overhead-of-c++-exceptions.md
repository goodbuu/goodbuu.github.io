---
title: The Actual Performance Overhead of C++ Exceptions
published: true
---

There are many claims about the performance overhead of C++ exceptions, but what is the reality?

## Throw vs. return

Benchmark: [Quick C++ Benchmarks](https://quick-bench.com/q/it0NASOaOz0lQqF7LLD5I7ZwGQg)

`HandleErrorWithThrow` throws when its argument is `0`; otherwise, it increments the argument. `HandleErrorWithReturn` follows the same logic but returns `false` instead of throwing. Inputs are generated with `std::uniform_int_distribution<>(0, 9)`, so the error case comes up 10% of the time.

At this rate, `HandleErrorWithReturn` is about 15x as fast as `HandleErrorWithThrow`.

In general, the cost of throwing and handling an exception comes mainly from three sources:

- Throwing an exception typically requires dynamic memory allocation for the exception object.
- Finding a matching catch block involves runtime type information (RTTI).
- The runtime unwinds the stack, frame by frame.

## Try/catch vs. direct call

Benchmark: [Quick C++ Benchmarks](https://quick-bench.com/q/1Z8G1MgJgFq30NY1g8_JrGHuxlo)

`NoActualException` throws only when its argument is `-1`. The inputs still range from `0` to `9`, so no exception is ever thrown. Because `NoActualException` is not declared `noexcept`, the call is potentially throwing. The first version wraps the call in a try/catch block, while the second calls it directly.

Both versions perform similarly.

The try/catch block adds no instructions to the hot path. However, exception-handling support can increase code size, which may put additional pressure on the L1 instruction cache.

## Conclusion

- Do not use exceptions for normal control flow.
- You generally do not need to worry about the runtime overhead of exception handling when no exception is thrown.
