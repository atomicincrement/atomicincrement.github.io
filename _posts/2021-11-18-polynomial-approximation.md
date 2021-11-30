---
layout: post
title:  "Breaking the AI sound barrier with Doctor Syn."
date:   2021-11-18
categories: maths
---

## Executive Summary

The problem facing the AI industry today is that many of the functions at
the heart of machine learning processes were written over forty years ago
when computer hardware and compiler technology were very different.

Our library, Doctor Syn, addresses this problem by using the three technologies
of SIMD, multithreading and autovectorisation. We achieve 30x or more speedups
over traditional libraries in C, C++, Rust and Fortran, without making the code
platform or language specific.

Doctor Syn's primary focus at present is to generate accurate polynomial
approximations to key functions important to the execution of many programs. You are probably familiar with many of the functions we are targeting:

| Rust Function | calculates |
|---------------|------------|
| f32/f64::sin      | $$\sin{x}$$ |
| f32/f64::cos      | $$\cos{x}$$|
| f32/f64::atan2      | $$\arctan{y/x}$$|
| f32/f64::exp      | $$e^x$$|
| f32/f64::ln      | $$\log{x}$$|

[See Wikipedia](https://en.wikipedia.org/wiki/C_mathematical_functions)

While improving these functions has a lot of value, we are currently focusing most of our effort on statistical functions such as:

| R Function | distribution | role | calculates |
|------------|--------------|------------|
| dnorm      | normal | pdf | $$\frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{1}{2}\left(\frac{x - \mu}{\sigma}\right)^2}$$|
| pnorm      | normal | cdf | $$\frac{1}{2}\left[1 + \operatorname{erf}\left( \frac{x-\mu}{\sigma\sqrt{2}}\right)\right] $$|
| qnorm      | normal | quantile | $$\mu+\sigma\sqrt{2} \operatorname{erf}^{-1}(2p-1)$$|
| rnorm      | normal | random | $$ \operatorname{qnorm}(\operatorname{runif}(i)) $$|

[See the R documentation](https://stat.ethz.ch/R-manual/R-devel/library/stats/html/Normal.html)

These functions are used extensively in finance and bioinformatics to perform statistical
inference, stochastic modelling, AI and Machine learning. For example, rnorm is a key part of many
MCMC algorithms and variational techniques, as well as a key part of monte carlo simulations, such as those that may be used to solve stochastic differential equations.

Using this library, combined with parallel iterators, we generate more efficient versions of

* Numpy
* R
* GNU Octave

and many others.

We have also targeted new architectures like Arm SVE which do not fit the X86 model.
We are working with the Isambard A64FX cluster to attempt to improve existing
algorithms.

This approach to function generation should fit perfectly with the A64FX's SVE
architecture as SVE has a variable length SIMD architecture which will run
the same binary on many different word-length machines. SVE requires
Autovectorisation to work effectively.

## The AI sound barrier

While great effort has been expended in key function optimization, current techniques are unable to efficiently utilize modern compiler technology such as auto-vectorisation and thread based parallelism. This leads to the situation in which users are forced to trade off accuracy and speed, with no implementation that is both *fast* and *accurate*.

[See this for an example of *accurate*](https://blog.sigplan.org/2021/08/26/high-performance-correctly-rounded-math-libraries-for-32-bit-floating-point-representations/)

[See this discussion on *fast* on Stack Overflow](https://stackoverflow.com/questions/18662261/fastest-implementation-of-sine-cosine-and-square-root-in-c-doesnt-need-to-b)

The result is usually a compromise and this has been the status quo for several
decades now. In fact there have been very few improvements to math libraries
since apart from hand-written assembler versions of older functions. This has
created a "sound barrier" that has yet to be broken where AI code is limited
to a certain performance. To go faster, you will need to buy expensive GPU
hardware that consumes enormous amounts of power and then your
benefit will be minimal.

## The challenge of vectorisation.

Existing functions will not vectorise primarily because:

* They are in shared or static libraries.
* They contain branches and look-up tables.

In the 1970's when many of these functions were written, this was the state of the art. Today however, this is a problem: with the advances of modern vectorising processors, these implementations are substantially less efficient than they could be.

There is no short and simple fix to this problem either, these problems are fundamental ones that preclude any efficient vectorisation. The only viable approach to this problem is with substantial and novel changes such as the one that Doctor Syn proposes.

## Autovectorisation

In the past we have implemented fast functions in assembler or even written assembler
to write functions in machine code. These days however, we often try to take a more "civilized" approach where possible. Assembler functions are fast, but difficult to read, difficult to improve, and difficult to generalize. Despite these problems, many such functions end up hanging around like a bad smell, and more are being produced by chip vendors for special architectures. Even excellent libraries like [Sleef](https://github.com/shibatch/sleef/blob/master/src/arch/helperavx512f.h) are done this way with machine specific intrinsics.

To solve the problems with these architecture specific assembly implementations, we try and write portable code; we wish our code to be able to run on both x86 architectures with SIMD as well as the new ARM SVE with variable sized registers. The way we achieve this is by trying to write code in such a way as that it will be automatically vectorised by modern compilers.

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

we simply write:

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

```C
#include <math.h>

void vector_sin(double *d, int len) {
    while (len--) {
        *d = sin(*d);
        ++d;
    }
}
```

[Godbolt](https://godbolt.org/z/TG8on7Mfd) for x86-64 clang gives:

```
.LBB0_7:                                # =>This Inner Loop Header: Depth=1
        vmovsd  xmm0, qword ptr [rbx + 8*rbp]   # xmm0 = mem[0],zero
        call    sin
        vmovsd  qword ptr [rbx + 8*rbp], xmm0
        vmovsd  xmm0, qword ptr [rbx + 8*rbp + 8] # xmm0 = mem[0],zero
        call    sin
        vmovsd  qword ptr [rbx + 8*rbp + 8], xmm0
        vmovsd  xmm0, qword ptr [rbx + 8*rbp + 16] # xmm0 = mem[0],zero
        call    sin
        vmovsd  qword ptr [rbx + 8*rbp + 16], xmm0
        vmovsd  xmm0, qword ptr [rbx + 8*rbp + 24] # xmm0 = mem[0],zero
        call    sin
        vmovsd  qword ptr [rbx + 8*rbp + 24], xmm0
        add     rbp, 4
        cmp     r14d, ebp
        jne     .LBB0_7
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

We tested both single and multi-threaded versions of these functions - easy
in Rust as it is a naturally multi-threaded language - on a four core X86 laptop.

|Library|Function|ns per iteration (smaller is better)|
|-------|----|----------------|
|Doctor Syn|`runif`|0.8|
|Doctor Syn|parallel `runif`|0.6|
|rand|`ThreadRnd::gen()`|5.1|
|rand|parallel `ThreadRnd::gen()`|2.1|
|R|`runif`|35.0|
|Numpy|`numpy.random.uniform`|35.0|
|C|`rand() * (1.0/RAND_MAX)` -O3|6.0|
|C++|`uniform_real_distribution` -O3|13.6|

So clearly, we do well against even the best Rust version and
much better (over 30 times better) than R and Numpy.

Moving to normal random number generation, we use the quantile (or probit) function
to shape the random variable:

```
fn qnorm(arg: fty) -> fty {
    let scaled: fty = arg - 0.5;
    let x = scaled;
    let recip: fty = 1.0 / (x * x - 0.5 * 0.5);
    let y: fty = (177186111.131545818686411653000483 as fty)
        .mul_add(x * x, -219058235.58919835 as fty)
        .mul_add(x * x, 117054121.857504129646289572504640 as fty)
        .mul_add(x * x, -35345955.68660036 as fty)
        .mul_add(x * x, 6623473.609141078534685775398250 as fty)
        .mul_add(x * x, -796318.1973069897 as fty)
        .mul_add(x * x, 61391.409088151006196662227193 as fty)
        .mul_add(x * x, -2938.7971360761 as fty)
        .mul_add(x * x, 83.911295471202339471921364 as fty)
        .mul_add(x * x, 0.012702493639562371692090 as fty)
        .mul_add(x * x, 1.856861340488065073103038 as fty)
        .mul_add(x * x, -0.626662948075053 as fty)
        * x;
    y * recip
}

/// Use qnorm to shape the uniform random number.
pub fn rnorm(index: usize) -> f64 {
    qnorm(runif(index) * 0.999 + 0.0005)
}

/// Parallel version in Rust.
#[target_feature(enable = "avx2,fma")]
unsafe fn test_par_rnorm(d: &mut [f64]) {
    do_par(d, |d| *d = rnorm(ref_to_usize(d)));
}
```

|Library|Function|ns per iteration (smaller is better)|
|-------|----|----------------|
|Doctor Syn|rnorm|2.4|
|Doctor Syn|parallel rnorm|0.9|
|rand_distr|Normal::sample()|6.9|
|rand_distr|parallel Normal::sample()|1.7|
|R|rnorm|65.0|
|Numpy|numpy.random.uniform|60.4|
|C++|`normal_distribution<double>` -O3|31.0|

So more than 60x speedup over the R an dpython versions
on a four core laptop and about 30x for C++.

## Future work

With an implementation of just two Doctor Syn functions, we have shown a significant performance boost
over even the best-in-class Rust distribution system. This excellent result on a toy example shows excellent promise for what the generalised **Doctor Syn** system is capable of. 

Work has started on support for ARM SVE, The Doctor Syn method also provides flexibility
in the accuracy of the solution it provides, and we are exploring super-accurate
versions of our functions by using larger sizes, table lookups or fixed point integer
arithmetic. Finally, we are working on the substantial task of full verification as
well as generation of R, python and Octave libraries. 

Atomic Increment is developing this technology in partnership with [Embecosm](https://www.embecosm.com). If you want to use this technology, get in touch. We are keen to develop industry
partnerships with companies who require extra performance in their machine learning
and computational processes.

andy@atomicincrement.com
jeremy.bennett@embecosm.com

