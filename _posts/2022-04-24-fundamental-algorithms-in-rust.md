---
layout: post
title:  "Fundamental Algorithms In Rust"
date:   2022-04-24 12:00:00 +0000
categories: algorithms
---
## Introduction

Computer science is much like Victorian Taxonomy. The models is Computer
scientists go into the field and return with algorithms and then organise
them in a system. Programmers, the people who actually write software, can
then choose from an array of algorithms to solve problems.

But in reality, a good programmer will create new algorithms to suit their
needs and this will usually be faster than trawling through academic papers
which are usually locked behind paywalls. You will find many novel algorithms
in open source code and many more are left to rot in corporate codebases.

Despite this, it is worth learning some basics. The Knuth books contain an
excellent collection of algorithms - many in the excercises - and the Cormen,
Leiserson, Rivest, Stein *Introduction to algorithms* is a good reference.

Today there are excellent articles on Wikipedia that will also help you through
your journey of discovery.

Here I am going to talk about a number of data stuctures and algorithms
with illustrations in Rust. Rust is a more modern programming language
that combines safety with performance.

*Note that the examples I give are not the most efficient implementations
as these are often quite complex.*

## Categories of data structures and algorithms

This is a list of some of the basic categories of data structures and algorithms.
I cover each of these in different sections.

* Sorting
* Containers
* Dynamic programming
* Greedy algorithms
* Graph algorithms
* Computation geometry
* String matching
* Reduction

Of course, there are many, many more; but we have to start somewhere.

## Practical considerations

Computers have changed quite a bit since the first algorithms books were
written. In particular computation now far outstrips the speed of the
memory and a typical machine may have hundreds or thousands of cpus.

Running multi-threaded code in a single process will generally outstrip
the performance of running multiple single threaded processes which has
beconme the norm. But programmers who can write multi-threaded code are
rare.

Using O-notation, common in many textbooks, will give you a guide to
how an algorithm works when the workload becomes infinite. Of course
there are no infinite workloads that are not entirely I/O bound and
today, we often ignore O-notation for more practical considerations.
For example, searching for an element in a vector is a perfect job
for a modern CPU and so storing data in vectors is often prefered
to more complex data types such as B-trees.

## O-notation

A traditional measure of algortithmic efficiency is the O-notation.

Technically if $$f(x) = O(g(x))$$ then

$$|f(x)| \le M g(x) \quad \text{ for all } x \ge x_0.$$

for some $$M$$.

For example if $$f(x) = O(N)$$ then

$$|f(x)| \le M N \quad \text{ for all } x \ge x_0.$$

for some $$M$$.

ie. the run time will grow linearly with a constant $$M$$.

If $$f(x) = O(N^2)$$ then

$$|f(x)| \le M N^2 \quad \text{ for all } x \ge x_0.$$

for some $$M$$.

ie. the run time will grow as the square of $$N$$.

O-notation is not the divine oracle that computer scientitsts
think it is, remember Andy's rule:

*The problem with theoretical computer science is that there
are no theoretical computers*.

## Sorting

### Insertion sort

Lets start with the classic insertion sort. This is usually taught
as the *dont do* of sorting algorithms, but when the arrays are short
it often outperforms other sorts.

The insertion sort picks the next value in the array and inserts
it into the values before it. This way the array is always sorted
up to a certain position. When we get to the end, the whole array
is sorted.

Notionally insertion sort takes $$O(N^2)$$ operations.


```rust
pub fn insertion_sort(values: &mut [i32]) {
    // For all values.
    for i in 0..values.len() {
        // Insert the current value into the start of the array.
        for j in (0..i).rev() {
            if values[j] < values[j+1] { break }
            values.swap(j, j+1);
        }
    }
}
```

### Bubble sort

Bubble sort notionally has a longer running time than insertion sort
but it has huge advantages for small N.


```rust
pub fn bubble_sort(values: &mut [i32]) {
    for i in 0..values.len() {
        for j in i+1..values.len() {
            if values[i] > values[j] { values.swap(i, j) }
        }
    }
}
```

Notionally bubble sort takes $$O(N^2)$$ operations but,
the run time is constant which means the loops can be unrolled like this:

```rust
pub fn bubble_sort_3(values: [i32; 3]) -> [i32; 3] {
    let [mut x, mut y, mut z] = values;
    if x > y { std::mem::swap(&mut x, &mut y); }
    if y > z { std::mem::swap(&mut y, &mut z); }
    if x > y { std::mem::swap(&mut x, &mut y); }
    [x, y, z]
}
```

This is much faster.

### Merge sort

The merge sort is a *divide and conquer* method that sorts
two sub-arrays and then merges them.

Merge sort is very easy to parallelise as each step can be run on
a different task.

Notionally, merge sort takes $$O(N log(N))$$ operations to run.

```rust
pub fn merge_sort(values: &mut [i32], tmp: &mut [i32]) {
    let len = values.len();
    if len < 4 {
        // Below a certain size, do an insertion sort.
        insertion_sort(values);
    } else {
        // Above a certain size, split into two and sort separately.
        let mid = len / 2;
        merge_sort(&mut values[0..mid], &mut tmp[0..mid]);
        merge_sort(&mut values[mid..len], &mut tmp[mid..len]);

        // Duplicate the values and merge them.
        tmp.copy_from_slice(values);
        values
            .iter_mut()
            .zip(itertools::merge(&tmp[0..mid], &tmp[mid..len]))
            .for_each(|(d, s)| *d = *s);
    }
}
```

### Quick sort

Quick sort is another divide and conquer algortithm that partitions
all the elements into three parts for a randomly chosen pivot value.

* Elements < pivot.
* Elements == pivot.
* Elements > pivot.

It is traditional to take the median of the start, middle and end
as the pivot and to use an unrolled bubble sort for the leaves.

Notionally, quick sort takes $$O(N^2)$$ operations to run, because
in the worst case, the pivot could always be chosen as the start or end
value.

In practice it is usually less than $$O(N log(N))$$ runtime.


```rust
pub fn median(mut x: i32, mut y: i32, mut z: i32) -> i32 {
    if x > y { std::mem::swap(&mut x, &mut y); }
    if y > z { std::mem::swap(&mut y, &mut z); }
    if x > y { std::mem::swap(&mut x, &mut y); }
    y
}

pub fn quick_sort(values: &mut [i32]) {
    let len = values.len();
    if len < 4 {
        // Below a certain size, do a bubble sort.
        bubble_sort(values);
    } else {
        // Guess a pivot value to partition the array.
        let pivot = median(values[0], values[len / 2], values[len-1]);

        // Partition into 3 parts [0..p] [p..q] and [q..len]
        let p = itertools::partition(values.iter_mut(), |&i| i < pivot);
        let q = itertools::partition(values[p..].iter_mut(), |&i| i == pivot) + p;

        // Sort the two unsorted partitions (no need to sort values[p..q]).
        quick_sort(&mut values[0..p]);
        quick_sort(&mut values[q..len]);
    }
}
```

## Containers

Containers in Rust encapsulate standard data structures made popular by
computer science books and papers.

We can use the standard containers such as `Vec` and `HashMap` out of the
box and the implementations are pretty decent. These containers work
for the majority of cases, and can be composited into novel
algorithms. For most programmers, it would be hard to beat them.

### Vectors

In Rust we have the standard vector to represent a growable array of
items. Internally, it contains a pointer and two sizes.

```
Vector (24 bytes)

pointer  len      capacity
------------------------
pppppppp llllllll cccccccc
```

The pointer points to some bytes in memory, the length tells us how
many we have and the capcity allows us to grow the array without allocating
more memory, which is a costly process.

A vector is nearly always the correct data structure to use to get the
best performance. More complex data structure exist, but they all require
multiple memory allocations and leave the memory in a fragmented state.

### Hash maps

A hash map maps a key (often a string) to a value.

