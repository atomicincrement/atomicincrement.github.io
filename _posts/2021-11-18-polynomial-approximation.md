---
layout: post
title:  "Making faster stats functions."
date:   2021-11-18
categories: maths
---

## Doctor Syn

Recently I have been working on a Rust library called "Doctor Syn", a play
on the excellent **syn** library in Rust and Russel Thorndyke's character
who is both a priest and smuggler on the Romney marshes.

Doctor Syn's primary focus at present is to generate accurate polynomial
approximations to functions of one variable. You are probably famililar with
many of the functions we are targeting:

| Rust Function | calculates |
|---------------|------------|
| f32/f64::sin      | $$\sin{x}$$|
| f32/f64::cos      | $$\cos{x}$$|
| f32/f64::atan2      | $$\arctan{y/x}$$|
| f32/f64::exp      | $$e^x$$|
| f32/f64::ln      | $$\log{x}$$|

But we are mostly focused on statistical functions such as:

| R Function | distribution | role | calculates |
|------------|--------------|------------|
| dnorm      | normal | pdf | $$\frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{1}{2}\left(\frac{x - \mu}{\sigma}\right)^2}$$|
| pnorm      | normal | cdf | $$\frac{1}{2}\left[1 + \operatorname{erf}\left( \frac{x-\mu}{\sigma\sqrt{2}}\right)\right] $$|
| qnorm      | normal | quantile | $$\mu+\sigma\sqrt{2} \operatorname{erf}^{-1}(2p-1)$$|
| rnorm      | normal | random | $$ \operatorname{qnorm}(\operatorname{runif}(i)) $$|

[See the R documention](https://stat.ethz.ch/R-manual/R-devel/library/stats/html/Normal.html)

These functions are used extensively in finance and bioinformatics to perform statistical
inference to discover the parameters of models. For example, rnorm is often used to calculate
the next step in the Metropolis-Hastings algorithm.

## Surely this a solved problem?

Whilst there are both fast or accurate versions of many of these functions, none of
them are able utilise modern compiler technology such as autovectorisation and
thread based parallelism efficiently. Also there are no fast *and* accurate versions
of these functions, so we usually have to pick one.

[See this for an example of *accurate*](https://blog.sigplan.org/2021/08/26/high-performance-correctly-rounded-math-libraries-for-32-bit-floating-point-representations/)

[See this discussion on *fast* on Stack Overflow](https://stackoverflow.com/questions/18662261/fastest-implementation-of-sine-cosine-and-square-root-in-c-doesnt-need-to-b)

If you really need to be accurate, it is best not to use floating point at all or to use
a larger floating point type than the one you are calculating. For example for **float16** use
**float32** to do the calculation and for **float32** use **float64**. For **float64** you will
be better off using fixed point integer arithmetic. Solutions that use table lookup are very
poor performers on 2020-era CPUs, but may get better as the **gather** instructions become
optimised in SIMD implementations.

## Autovectorisation

In't old days we would write our fast functions in assembler or even write assembler
to write our functions in machine code. But we like to think that we are more civilised
now. Such functions were hard to read and impossible to improve. Many such functions
hang around like a bad smell and more are being produced by chip vendors for special
architectures. Excellent libraries like [Sleef](https://github.com/shibatch/sleef/blob/master/src/arch/helperavx512f.h) are done this way with machine specific intrinsics .

But to make our code portable, for example to run on the new ARM SVE with variable sized registers,
we need a wy of writing regular C or Rust code and having the compiler convert our loops into
SIMD code. We try to avoid writing SIMD code ourselves, these days, instead we make sure that
our code will vectorised on modern compilers.

So instead of:

```rust
fn inc_doubles_simd(x: &mut [f64]) {
    for x in x.chunks_exact(4) {
        // Some complex SIMD code
    }
    for x in x.chunks_exact(4).remainder() {
        *x += 1;
    }
}
```

we can write:

```rust
fn inc_doubles_scalar(x: &mut [f64]) {
    for x in x {
        *x += 1;
    }
}
```

## Vectorises or not?

Vectorisers are fickle beasts however. If the wind blows in the wrong direction, the compiler
will often fail to vectorise or worse - vectorise in the IR and then convert the vector
operations to a long series of library calls.

For example, the following rather innocent function, which absolutely should be vectorisable,
converts itself into a series of function calls:

```rust
pub fn vector_sin(d: &mut [f64]) {
    d.iter_mut().for_each(|d| *d = f64::sin(*d))
}
```

[Goldbolt](https://godbolt.org/z/x4jr6MdM3) gives:

```
.LBB0_6:
        vmovsd  xmm0, qword ptr [r15 + 8*r12]
        vmovsd  qword ptr [rsp], xmm0
        vmovsd  xmm0, qword ptr [r15 + 8*r12 + 8]
        call    r14
        ... and 7 more calls like this.
        vunpcklpd       xmm0, xmm0, xmmword ptr [rsp]
        vmovups xmmword ptr [r15 + 8*r12 + 48], xmm0
        add     r12, 8
        add     rbp, 4
        jne     .LBB0_6
```

Each call will take several hundred cycles.

## Making library functions that vectorise

So how can we make code snippets like the above
vectorise? One way is to get LLVM, the backend to Rust and
many other languages, make calls to "vector" versions
of the library. Indeed LLVM has support for vector versions
of sin, cos etc.

This is a good idea except for one major flaw. Any
calls to libraries that do not inline will suffer from
latency issues.

To understand why this is the case, you have to understand
latency in instructions.

Say we have some code like this:

```rust
pub fn latency(x: f32) -> f32 {
    let x = x + 1.0;
    let x = x * 20.0;
    let x = x * x;
    x
}
```

```
example::latency:
        vaddss  xmm0, xmm0, dword ptr [rip + .LCPI0_0]
        vmulss  xmm0, xmm0, dword ptr [rip + .LCPI0_1]
        vmulss  xmm0, xmm0, xmm0
        ret
```

Here we execute three instructions, but their latency
is usually more than one cycle - four is common.

This means that there is a long wait between the instructions
like this:

```
example::latency:
        vaddss  xmm0, xmm0, dword ptr [rip + .LCPI0_0]
        # wait of 3 cycles
        vmulss  xmm0, xmm0, dword ptr [rip + .LCPI0_1]
        # wait of 3 cycles
        vmulss  xmm0, xmm0, xmm0
        # wait of 3 cycles
        ret
```

Assuming the call is invisible, this means we get a throughput of
`1/(3 x 4) = 1/12` operations per cycle or `4/(3 x 4) = 1/3` for
vectors of 4 elements. This is four times slower than we
can achieve with inline code, which can unroll loops to get
more throughput, a technique called **interleaving**.

A better function would look like the following using dense groups
of four vector instructions with a throughput of `8 x 4 / 12 = 8 / 3`
operations per cycle.

```
example::latency2:
        vaddps  ymm0, ymm0, dword ptr [rip + .LCPI0_0]
        vaddps  ymm1, ymm1, dword ptr [rip + .LCPI0_0]
        vaddps  ymm2, ymm2, dword ptr [rip + .LCPI0_0]
        vaddps  ymm3, ymm3, dword ptr [rip + .LCPI0_0]
        vmulps  ymm0, ymm0, dword ptr [rip + .LCPI0_1]
        vmulps  ymm1, ymm1, dword ptr [rip + .LCPI0_1]
        vmulps  ymm2, ymm2, dword ptr [rip + .LCPI0_1]
        vmulps  ymm3, ymm3, dword ptr [rip + .LCPI0_1]
        vmulps  ymm0, ymm0, ymm0
        vmulps  ymm0, ymm0, ymm0
        vmulps  ymm0, ymm0, ymm0
        vmulps  ymm0, ymm0, ymm0
        ret
```

Library functions also make things bad for themselves by introducing
branching. To get better accuracy, they divide the domain
of a function - for example $$[-\pi, \pi]$$ for $$\sin(x)$$ into many small
parts. This is often done using a `switch` statement which will not vectorise.
Alternatives include using a lookup table of coefficients but many CPUs have not yet
implemented an efficient `gather` operation which can do table lookups
in reasonable time. The exception to this is GPUs, which do commonly
have efficient `gather` but these are likely to hurt the cache performance
unless you use non-temporal loads and stores.

So a good maths function will often sacrifice a little accuracy - one or two bits
to simplify the execution, but this is a choice that must be made. In the humble
opininion of the author, if you want more accuracy, use a larger number type
such a f64 for f32 and i64 (fixed point) for f64 calculations. This enables
you to calculate sin and cos as a single polynomial instead of dividing
it into quadrants.

## The anatomy of a vectorisable maths function

Maths functions have many uses. In biology and finance, we usually want high throughputs
of vector data, in games we usually want a low latency function at the expense of
throughput, some uses require high accuracy and others require high performance while
still others may operate over a limited domain.

These design choices are usually "concreted in" in library functions. Even if we
know the input of a function will never have NaNs, we will still do a NaN check.
Likewise the input domain may be limited.

To improve this we need to offer design choices to the users of functions.
A typical function looks like this:

```rust
fn my_custom_function(x: f32) -> f32 {
    if let Some(nan) = domain_checking(x) {
        return nan;
    }
    let r = domain_reduction(x);
    let s = domain_scaling(r);
    let p = pole_elimination(s);
    let x = polynomial_approximation(p);
    let y = domain_reconstruction(x);
    y
}
```

### Domain checking

`domain_checking()` is the process of testing the incomming
parameters against known bounds. For example `exp(x)` has an
operating range of roughly $$x < 710$$ beyond which we overflow
for values $$x < -710$$ we will underflow and this needs to be
handled.

Domain checking is optional if you know that the input range
is always satisfied, for example when calculaing `logsum`
or `softmax` in neural network execution.

### Domain reduction

`domain_reduction()` will reduce the operating domain of a function
down to a smaller range. For example $$\sin(x) = \sin(x + 2\pi)$$
so we can reduce x to a smaller range modulo $$2\pi$$

For `exp(x)` we calulate $$2^x$$ and separate into integer and fractional
parts. The integer part becomes the exponent and the fractional
part goes through the polynomial to become the mantissa.

### Pole elimination

`pole_elimination()` uses a rational denominator to eliminate
poles (infinites) in functions. This increases the accuracy
for functions like $$\tan(x)$$ which has infinites at $$\pm\frac {\pi} {2}$$.
We do this by dividing the result by $$(x-a)(x-b)...$$ where $$a$$ and $$b$$ are
the locations of the poles. We can add more poles outside the domain
to "straighten out" the function and reduce the size of coefficients in
the next step.

### Polynomial approximation

`polynomial_approximation()` calculates a polynomial approximation
of a function in a limited domain. If the domain is very small, a Taylor
series can be used, but in **Doctor Syn** we use Newton polynomials
to approximate any function, even those without Taylor series.

The problem with Taylor series is that they approximate a function
accurately only at one point, but a carefully chosen Newton polynomial
will give accurate results over a wide range. We can improve on the
polynomial using the Remez algorithm, but it can be highly unstable
and usually the benefit is small.

Polynomial approximations can be chose to make the result 100% accurate
in certain places. For example $$\sin(\frac{\pi}{2})$$ should be exact.

### Domain reconstruction

Usually when doing the domain reduction and pole elimination, we need
to map back into the final domain afterwards. For example when
calculating $$\exp(x)$$ we need to reconstruct the result from
exponent and mantissa or in a quadrant form $$\sin(x)$$ where
we calculate $$\sin(x)$$ and $$\cos(x)$$ we need to select the
quadrant.

In the case of poles, we divide by our "pole polynomial" to
reintroduce the poles we eliminated.

