`flow` - C++ template library for overflow-free integers
========================================================

Introduction
------------

Arithmetic overflow is a major source of concern for program correctness (computes the right result), security (prevents unauthorized access), robustness (doesn't crash), and safety (doesn't cause bodily harm). Yet programming languages like C++ provide no guardrails against them. Even a simple expressions like `b = a + 1` can result in `b` being (much) smaller than `a` and it's considered up to the programmer to **prevent**, **detect**, or **avoid** this! Moreover, while the result of unsigned integer addition is well-defined, signed addition overflow (or underflow) is [undefined behavior](https://en.cppreference.com/w/cpp/language/operator_arithmetic#Overflows).

```C++
abort_if_overflow<expression>(a, b);  // prevention
c = expression(a, b);
risky_operation(c);
```

**Preventing** overflow means testing all the inputs to an expression to determine whether overflow will or could happen and taking appropriate action against it (returning an error, throwing an exception, or aborting the program). For example computing `a * a` is safe against overflow if `a` is of type `int32_t` and `a < 1000`. But this is a highly conservative range. The actual limit that's safe is `a <= 46340`. This is not a widely known value, despite the very simple single-operation expression. This immediately illustrates that prevention through accurate checking of the ranges of input values is a difficult task. For even slightly more complex expressions it can become downright impossible not to make mistakes. And yet *conservative* prevention is a very powerful and often *high performance* option to eliminate most risks of overflow. If all the values used by your program or API are checked early on to be smaller than say 1000, the chances of having an expression which may overflow are very slim. But are you feeling lucky?

```C++
c = expression(a, b);
abort_if_overflow(c, a, b);  // detection
risky_operation(c);
```

**Detecting** overflow typically relies on results being way off from the expected range for the given input. Like addition of two positive integers resulting in a negative value, or a value smaller than either input. Note that for signed addition this is implementation-dependent. The hardware or runtime library may perform saturating addition, for example, for which detection requires performing a subtraction. Or the hardware may provide flags that indicate overflow, or raise a signal. Suffice to say this isn't any easier than prevention.

Note that *prevention* and *detection* can be considered two sides of the same coin. Detection serves no purpose if it doesn't also *prevent* a harmful subsequent operation from happening (like accessing memory out-of-bounds). And prevention can be performed by executing a copy of the expression and *detecting* whether overflow happened in this copy, before executing the real expression.

```C++
big_c = expression(a, b);  // avoidance
risky_operation(big_c);
```

**Avoiding** overflow circumvents the problem altogether by using types with sufficient precision. Modern CPUs have full 64-bit integer data paths. We just store 8-, 16-, 32-, or 64-bit portions of the results to memory for the purpose of packing and alignment. But to avoid overflow in operations between 32-bit input values we pretty much just have to store the full 64-bit output computed by the hardware, which C++ only allows by extending the input to 64-bit beforehand. One extreme counterexample where that isn't enough is a left shift by more than 32 bits.

The approach of avoiding overflow entirely can also be observed in higher-level languages such as Python. And libraries exist for C++ which emulate support for fully arbitrary precision integers. But this typically comes with the downside of massive loss of performance. However, it is the *only* downside; widening types avoids overflow altogether. When the result type is 64-bit or less, and this is fully specialized for (i.e. no dynamic allocation happening), there would be no performance penalty.

Lastly, note that the techniques of prevention, detection, and avoidance of overflow can all be mixed and matched. Long expressions and multiple statements can be broken up or combined, with inputs, outputs, and intermediate results all being checked, extended, or truncated and checked again in various places. Programmers have a ton of unclear tradeoffs to make, and it has an overall net negative impact on productivity, maintenance, readability, and root cause analysis.

Design
------

The objective of `flow` is to achieve best practices in overflow mitigation effortlessly. The main idea is to use compile-time analysis to assist developers in determining where to take which measures.

First, input value range reduction is encouraged and communicated to the library at compile-time by offering integer types with single-increment bit widths. For example `int_t<19>` is logically 19 bits wide and accepts values in the range of [-2<sup>18</sup>, 2<sup>18</sup> - 1] on conversion from a wider type. Values outside of this range cannot exist, with out-of-range conversion resulting in an empty `flow::optional<>` (analogous to `std::optional<>`), and thus a large level of *prevention* of overflows is achieved.

Secondly, arithmetic operations return their full-width result. For example a left-shift of `int_t<19>` by a `uint_t<3>` is known to fit in `int_t<26>` at compile-time. This system achieves overflow *avoidance* at often zero cost or otherwise minimal overhead.

Thirdly, conversion back to more narrow C++ standard types again returns an `flow::optional<>` which is empty if the value could not be represented, thereby forcing *detection* of overflows.

The design choice of using bit width to represent the valid range of integers, instead of an upper and/or lower limit, is a practical one. C++ programmers already understand the concept of type width in bits very well, and thinking in power-of-two range increments comes naturally. For logical value ranges that are not a power-of-two (minus one), often a bit of leeway doesn't hurt. `flow` encourages the use of e.g. `int_t<16>` for values in the range [0, 1000] where otherwise an `int32_t` or equivalent might have been chosen instead of an `int16_t`, because of the explicit check on construction. As long as it compiles without unexpectedly requiring later truncation of results there is no performance impact of choosing slightly larger bit widths. Also, requiring the full range of [-46340, 46340] for the above mentioned example of squaring a value and fitting the result in `int32_t` is deemed exceedingly rare; instead `int_t<16>` with a range of [-32768, 32767] should suffice for practical scenarios. Last but not least, it's a lot less typing and easier to read than something like `int_t<-46340, 46340>`. But there's always the option of adding such exact-ranged types later.

For completeness the library also offers narrowing of types through saturation or truncation. Both these options need careful deliberation whether they achieve the desired results. For example truncation can be acceptable for keeping array accesses within bounds if and only if accessing the wrong element isn't harmful, while e.g. saturation can be disastrous when 100,000 elements need to be allocated and it got reduced to 65,535 on saturated narrowing to `uint_t<16>`. But at least the library makes these operations very explicit instead of implicit like for basic C++ types. Also, more narrow types cannot be "truncated" to wider types; the presence of a truncation function means truncation *is* taking place and demands attention (unlike C++ where a type cast gives no indication of widening or narrowing and there can be a false sense of safety when many widening or redundant casts obscure a dangerous narrowing).

It's important to note that `flow` is *not* an arbitrary precision library. 64-bit is the largest type. Multiplication returning a `[u]int_t<>` is only defined for types that combine into 64-bit or less. Otherwise a `flow::optional<>` is returned.

All of this heavily relies on C++20 concepts to only allow valid widths and operations, and have readable error messages otherwise.

`flow::optional<>` has a few differences compared to `std::optional<>`: obtaining the numeric value with `operator*` will trigger an `assert()` in debug builds which verifies that it was previously converted to a `bool` (as part of checking whether the value exists, i.e. no overflow took place), or `has_value()` was called. It is the developer's responsibility to take appropriate action on overflow. There is also a `throw_if_no_value()` member and `abort_if_no_value()` member as a short form for two common response actions. `value()` throws `bad_optional_access` if no value is present, just like `std::optional<>`, so it's equivalent to the more explicit sequence of `throw_if_no_value()` and access through `operator*`. Note the latter is also more efficient than `value()` when accessing the same value multiple times in a loop by hoisting the check out of the loop.

---
<sub><sup>Copyright 2023 Nicolas Capens</sup></sub>
