---
layout: post
title:  "Making faster stats functions."
date:   2021-11-18
categories: maths
---

## Doctor Syn

Recently we have been working on a Rust library called "Doctor Syn", a play
on the excellent **syn** library in Rust and Russel Thorndyke's character
who is both a priest and smuggler on the Romney marshes.

Doctor Syn's primary focus at present is to generate accurate polynomial
approximations to functions. You are probably famililar with
many of the functions we are targeting:

| Rust Function | calculates |
|---------------|------------|
| f32/f64::sin      | $$\sin{x}$$ |
| f32/f64::cos      | $$\cos{x}$$|
| f32/f64::atan2      | $$\arctan{y/x}$$|
| f32/f64::exp      | $$e^x$$|
| f32/f64::ln      | $$\log{x}$$|

[See Wikipedia](https://en.wikipedia.org/wiki/C_mathematical_functions)

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
the next step in the Metropolis-Hastings algorithm or to simulate option pricing.

Using this libary, combined with parallel iterators, we can generate more efficent versions of

* Numpy
* R
* GNU Octave

and many others.

We can also target new architectures like Arm SVE which do not fit the X86 model.
We are working with the Isambard A64FX cluster to attempt to improve existing
algorithms.

## Surely this is a solved problem?

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

Existing functions will not vectorise primarily because:

* They are in shared or static libraries.
* They contain branches and look-up tables.

These things were not a problem in the 1970's when many of these functions
were written, but today this is a problem. So we can't just inline the
legacy functions and hope that they will vectorise because they are
not designed to do so.

## Autovectorisation

In the past we would write our fast functions in assembler or even write assembler
to write our functions in machine code. But we like to think that we are more civilised
now. Such functions were hard to read and impossible to improve. Many such functions
hang around like a bad smell and more are being produced by chip vendors for special
architectures. Excellent libraries like [Sleef](https://github.com/shibatch/sleef/blob/master/src/arch/helperavx512f.h) are done this way with machine specific intrinsics .

But to make our code portable, for example to run on the new ARM SVE with variable sized registers,
we need a way of writing regular C or Rust code and having the compiler convert our loops into
SIMD code. We try to avoid writing SIMD code ourselves, these days, instead we make sure that
our code will vectorised on modern compilers.

So instead of something like:

```rust
use std::arch::x86_64::*;

pub fn inc_doubles_simd(x: &mut [f64]) {
    unsafe {
        let one = _mm256_broadcast_sd(&1.0);
        for x in x.chunks_exact_mut(4) {
            let a = _mm256_loadu_pd(&x[0] as *const f64);
            let b = _mm256_add_pd(a, one);
            _mm256_storeu_pd(&mut x[0] as *mut f64, b);
        }
        for x in x.chunks_exact_mut(4).into_remainder() {
            *x += 1.0;
        }
    }
}
```

[Godbolt](https://godbolt.org/z/87eeadoej)

we can simply write:

```rust
fn inc_doubles_scalar(x: &mut [f64]) {
    for x in x {
        *x += 1;
    }
}
```

[Godbolt](https://godbolt.org/z/qG5asYMGn)

This is much easier to read, works on all known hardware without modifications and
does not specify a vector size, which might be variable.

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

Library functions make things bad for themselves by introducing
branching.
So even if we can inline the function, the functions will not vectorise.
To get better accuracy, they divide the domain
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
you to calculate functions like sin and cos as a single polynomial instead of dividing
it into sections.

**Doctor Syn** generates functions that are free of complex control flow
which would inhibit vectorisation. The functions are all available as source code
allowing inlining and as a result the chance of them inlining is much increased.

## Example - sampling from the normal distribution.

We tested some of our generated functions against one of the best stats distribution
libraries in the Rust world - [rand_distr](https://docs.rs/rand_distr/0.4.2/rand_distr/).

Combined with [rayon](https://docs.rs/rayon/1.5.1/rayon/) the parallel execution library
this would have been the best choice for monte carlo experiments.

We started with a uniform random number generator, based on a
[xorshift](https://en.wikipedia.org/wiki/Xorshift) hash and tested this against
rust's [ThreadRng](https://docs.rs/rand/0.8.4/rand/fn.thread_rng.html).

By using a hash of an integer index instead of a sequence, we are able to
    parallelise random number generation.

```
pub fn runif(index: usize) -> f64 {
    let mut z = (index + 1) as u64 * 0x9e3779b97f4a7c15;
    z = (z ^ (z >> 30)) * 0xbf58476d1ce4e5b9;
    z = (z ^ (z >> 27)) * 0x94d049bb133111eb;
    z = z ^ (z >> 31);
    from_bits((z >> 2) | 0x3ff0000000000000_u64) - 1.0
}
```

|Library|Function|ns per iteration (smaller is better)|
|-------|----|----------------|
|Doctor Syn|runif|0.8|
|Doctor Syn|parallel runif|0.6|
|rand|ThreadRnd::gen()|5.1|
|rand|parallel ThreadRnd::gen()|2.1|
|R|runif|20.0|
|Numpy|numpy.random.uniform|13.0|

So clearly, we do well against even the best Rust version and
much better (over 20 times better) than R and Numpy.

Moving to normal random number generation, we use the quantile (or probit) function
to shape the random variable:

```
fn qnorm(arg: fty) -> fty {
    let scaled: fty = arg - 0.5;
    let x = scaled;
    let recip: fty = 1.0 / (x * x - 0.5 * 0.5);
    let y: fty = (from_bits(4730221388428958202u64))
        .mul_add(x * x, -from_bits(4731626383781768040u64))
        .mul_add(x * x, from_bits(4727627778628654481u64))
        .mul_add(x * x, -from_bits(4720012863723153492u64))
        .mul_add(x * x, from_bits(4708869911609092829u64))
        .mul_add(x * x, -from_bits(4695087533321972728u64))
        .mul_add(x * x, from_bits(4678670384600451257u64))
        .mul_add(x * x, -from_bits(4658680898319303328u64))
        .mul_add(x * x, from_bits(4635605149421499302u64))
        .mul_add(x * x, from_bits(4578476110820645018u64))
        .mul_add(x * x, from_bits(4611041379213747643u64))
        .mul_add(x * x, -from_bits(4603819697584151827u64))
        * x;
    y * recip
}
```

|Library|Function|ns per iteration (smaller is better)|
|-------|----|----------------|
|Doctor Syn|rnorm|2.4|
|Doctor Syn|parallel rnorm|0.9|
|rand_distr|Normal::sample()|6.9|
|rand_distr|parallel Normal::sample()|1.7|
|R|rnorm|60.0|
|Numpy|numpy.random.uniform|31.9|

So more than 60x speedup over the R version
on a four core laptop and about 30x for numpy.

## Work to do

Whilst we clearly have a significant performance boost
over even the best-in-class Rust distribution system,
we still have some work to do.

To prove the system, we need to do considerable verification
work on the methods as well as generate R, python and GNU octave
libraries.

There are many other functions which will benefit from the
generalised system that **Doctor Syn** supports.

We should also look at super-accurate versions of these functions
by using larger sizes, table lookups or fixed point integer
arithmetic.

Work has started on support for ARM SVE, with work on the Isambard
A64FX cluster underway.