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

## Here be dinosuars.

Vectorisers are fickle beasts however. Great personalities with rather conservative views
dominate the open source compiler world and to them "correctness" is the most important thing,
whatever that may be - everyone disagrees. As a result, compilers often fail to vectorise innocent looking
loops resulting in drastic code slowdowns that are hard to diagnose.

For us, we just want our code to not take hours to run and consume a small nation's worth of power
in doing so and don't care much if two computer architectures get a different result in the 52nd bit
after all floating point is approximate after all and those who forget that will pay the price.

A classic example of this is **fused multiply add** which multiplies two numbers and
accumulates the result. There are special instructions on almost every CPU that will do this
and the result will not only be faster but more accurate also.

In Rust we can use the `.mul_add()` function on `f64` and `f32` types to do this explictly,
even in stable Rust. When calculating functions we get both more accuracy and more performance,
*win win!*

