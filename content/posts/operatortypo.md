+++
title = "Do as I meant, not as I wrote"
date = "2025-11-23T19:01:37Z"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
cover = ""
tags = ["C++"]
keywords = ["", ""]
description = "How a single-character typo corrupted a trading engine with no compiler errors... "
showFullContent = false
readingTime = false
hideComments = false
+++

### The world's worst copy assignment
I recently experienced a bug so nasty it made me reconsider my proclaimed "passion" for C++. Working on an implementation of a limit orderbook, I had implemented a `cancel` method, simply removing an `Order` object from a `std::vector` using `erase`.
```cpp
// ...
auto it = _orders_by_id[order_id]; 
orders.erase(it); 
// Note: This is a "Naive Vector" implementation for a future project
//       comparing orderbook architectures 
//       Iterator invalidation is part of that analysis! 
// ...
```
But, the `Order` was never removed. In fact, regardless of which `order_id` I was attempting to remove, it would remove **only** the last order. After stepping through the function in `gdb` more times than I care to admit, I checked my implementation of the `Order` class - and found this abomination.
```cpp
class Order {
// ...
    bool operator=(const Order& other) {
        // If two orders have the same id, they are the same order
        return id == other.id;
    }
// ...
}
```
At a glance, it appears to be a normal equality comparison operator - but closer inspection of the signature reveals this is in fact the **copy assignment operator!**. I happened to hastily leave off the extra '=', thus turning copy assignment into a **no-op** that returns `bool`. 

`erase` works by shifting down all elements after the deletion point, filling the gap left by the deleted item, before shrinking the vector by one. Because I had specified the copy assignment operator, the move assignment operator was thus deleted, forcing the vector to fall back on my broken copy assignment. The shifting became a no-op, and the poor order at the end of the list was truncated. The erroneous `bool` return value of my copy assignment was silently discarded, and the compiler moved on as if everything were normal. [See an example of my bug on Compiler Explorer](https://godbolt.org/z/367hacb1d)

After discovering this bug, I did what any sane C++ developer would do and direct my rage towards the compiler instead of accepting I was in the wrong. Why was I allowed to do this in the first place? Why did the code even compile with a clearly incorrect copy assignment operator? **Why does C++ not enforce the return type for operators?** In exploring these questions, I discovered exactly why this apparent anti-pattern actually allows for **incredibly smart optimisation techniques**, and enforcing operator return types would be a **terrible idea**. 

### Operators are pure syntactic sugar for function calls
Unlike many other languages, symbols such as `+`, `-`, `=`, `==` in C++ are **aliases to function calls**; `a + b` compiles to an AST node resembling `a.operator+(b)`. Operators play by the rules of function signatures just like any other function. A function signature is defined by: the name of the function, the number of parameters, and type of parameters. Notably, the **function's return type does not make this list** (which is why you cannot overload a normal function on return type alone). So the compiler can enforce `operator=` to take exactly one parameter of type `const T&`, but it does not enforce the return type. This anarchy actually allows us to build things that are impossible in safer languages.

### Use cases for "wrong" operator overloads
#### Expression templates
If C++ decided to enforce strict return types, making `operator=` always return `T&` and `operator+` always return `T`, it would destroy one of the most powerful optimisation techniques in math - and my latest obsession: **expression templates**.

Consider we are writing a linear algebra library, and we'd like to add 3 matrices.
```cpp
Matrix a, b, c;
Matrix result = a + b + c;
```
In a return-type-constrained world, `(a + b)` would first allocate a temporary `Matrix`, looping through rows and columns to calculate the sum. Then `(a + b) + c` would allocate `result`, again looping, before destroying the temporary `Matrix`. So 2 allocations and 2 loops for a simple addition - unacceptable in low-latency systems.

**Expression templates** provide a smart solution to this - and crucially, they depend on the flexibility of operator return types. In libraries like *Eigen*, `Matrix operator+` does not return a `Matrix`, but a lightweight helper object `Sum`.
```cpp
// Simplified expression template "Sum"
template <typename L, typename R>
struct Sum {
    const L& left;
    const R& right;
    // Keeping it simple for the blog - In production, this may cause dangling
    // references to temporaries!

    // This is only computed when required
    double operator[](int i) const {return left[i] + right[i]; }
};
template <typename L, typename R>
Sum<L, R> operator+(const L& l, const R& r) {
    // operator+ returns this helper, not the actual "result"
    return Sum<L, R>{l, r};
}
```
Now, our `Matrix` addition actually performs no addition - and no allocation. Instead the operators produce a nested structure which looks like `Sum{Sum{a, b}, c}`. The expression is only **evaluated when it is assigned to `result`**. As well as avoiding the unnecessary allocation, the compiler can also work its magic on this new sum type...
##### Loop Fusion and Heap Efficiency
As well as avoiding an allocation, the `Sum` expression template allows the compiler to generate SIMD instructions, and improves heap efficiency. Take the simple example of a `Matrix` addition with no expression template:
```cpp
Matrix result = a + b + c;
```
The CPU is condemned pass over the same memory twice.
- Pass 1: Load `a[i]`, load `b[i]`, add, write temporary result.
- Pass 2: Read temporary result, load `c[i]`, add, write to `result`.

When `operator+` returns a nested `Sum`, the computation only happens in the final assignment. The compiler is able to inline this, effectively turning the process into:
```cpp
for(int i = 0; i < N; ++i) {
    a[i] + b[i] + c[i]
}
```
The CPU can then:
- Load `a, b` and `c` directly into registers
- Sum them **instantly**
- **Write `result` once**

Don't just take my word for it! Check out the example on [Compiler Explorer](https://godbolt.org/z/zbcjodK53).

First look at the output for `SimpleVecSum`. It's huge! Notice the `operator new` and `operator delete` on lines 20, 88, 181, 196 and 304 (of compiler output). There's even a `memset` lurking on line 30. All this heap management vastly slows down the function.

Now look at `FastVecSum`. It's about a third of the size, and there are precisely zero heap allocations. But `clang` has done something even smarter here. Look at lines 324-336:
```asm
sub    r9, rdi
cmp    r9, 128
setb   r9b
; ... repeats for vectors b and c ...
or     r9b, r10b
je     .LBB1_5
```
Clang is **checking if the vectors overlap at runtime**. It wants to use 32-byte wide AVX registers to perform the sum, but if the vectors overlap, writing to the destination might overwrite data the source hasn't read yet. The seemingly magic number `128` actually indicates that the compiler wants to unroll the loop 4 times (8 bytes per double x 4 doubles per register x **4 unrolls** = 128). The `sub` instruction calculates the distance between vector `a` and result. If the distance is less than `128`, it sets the `r9b` flag - indicating that `a` and `result` overlap. If the flag is set, it falls through to `.LBB1_2`, full of slower - but safer - scalar instructions (`vaddsd`,`vmovsd`).

However, if the flag is **not** set, we see beautiful SIMD logic. Look at `.LBB1_8`, full of "packed double" SIMD instructions on 32-byte `ymm` registers. `SimpleVecSum` is prehistoric in comparison - all enabled because of proxy types, and ultimately, the freedom of operator overloading.
#### Embedded DSLs
What if we wanted to completely redefine the role of `operator+` and `operator=`. In not all cases does `=` mean assignment. Take a **constrained optimisation solver** for example. Naturally, you may want to define a constraint. In a rigid world, this may look like as follows:
```cpp
Solver.add_constraint(Variable{"x"}, Operator::EQUALS, 10);
```
While this makes the most sense when working **within the regular world of objects, allocation, and assignment**, we can hijack our operators to let the following code compile:
```cpp
Symbol x{"x"};
Solver.add_constraint(x = 10);
```
So in this case, `operator=` doesn't have **anything to do with assignment**, it will perhaps produce a `Constraint` object or similar.
### Best of Both Worlds
So it seems the ability to produce lazily evaluated computations and languages-within-languages come at the expense of tormenting beginners who mistake `=` for `==`. Luckily, C++20's `concepts` introduce a way to enforce **semantic intent** without sacrificing **syntactic freedom**
```cpp
#include <concepts>
template<typename T>
concept CopyAssignmentActuallyWorks = requires(T a, T b) {
    { a = b } -> std::same_as<T&>;
};

// Now the compiler will give some "strong guidance" if I mess up
struct Order {};
static_assert(CopyAssignmentActuallyWorks<Order>);
```
While this is overkill for a simple class (and `std::assignable` does exist), the point is that modern C++ allows us to shift mental capacity used by thinking about safeguards and valid states onto the compiler.
### The code (unfortunately) does exactly what you wrote
My one-character typo was permitted based on the assumption that I clearly had a brilliant reason to do so. GCC followed what I **wrote**, not what I **meant** - an effective contract. The architect is given complete freedom and trust by the builder, allowing the builder to build in **the best and most efficient way possible**; we just have to make sure we don't **accidentally design a house with no doors.**

Daniel Kirby, November 2025

