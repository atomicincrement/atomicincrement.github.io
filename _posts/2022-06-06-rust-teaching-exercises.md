---
layout: post
title:  "Rust teaching exercises"
date:   2022-06-06 14:12:06 +0000
categories: rust
---
The following are a collection of exercises to teach basic Rust
coding. Worked answers are at the end of this post.

## Exercise 1 Basic types

The variable `idx` indexes an array. Currently it is an 
`i32`, a signed 32 bit integer. Replace the type below
with the correct one to index the array.

### Bonus question:

What would happen if the index was 10?

```
fn main() {
    let array = [1, 2, 3];
    let idx : i32 = 1;
    println!("array[1]={}", array[idx]);
}
```

## Exercise 2a structs

Write code in main to construct each of these structures.

### Bonus question

How do we debug print these structures? (hint #[derive(...)])

```
struct CStruct {
    i: i32,
}

struct NewType(i32);

struct TupleStruct(i32, f32);

struct UnitStruct;

fn main() {
    // let cstruct = ...

    // Bonus question
    // println!("{:?}", cstruct);
}

```

## Exercise 2b enums

Construct all the variants of this enum.

### Bonus question

How many bytes does `MyEnum` take on the stack?
Is it different for each variant?

```
enum MyEnum {
    Unit,
    NewType(i32),
    CStruct { i: i32 },
    Tuple(i32, f32),
}

fn main() {
    // let unit = ...

}
```

## Exercise 3a vectors

Complete the following function and push
the integers 1, 2 and 3 into the vector.

Print the vector when done.

Why does push not work at first?

### bonus questions

How many bytes does the vector take:

* On the stack.
* On the heap.

What happens when we clone a Vec?

How can we push 3 items with one call?

```
fn main() {
    // What needs to change here?
    let myvec = Vec::new();

    // push 1 onto myvec
    // push 2 onto myvec
    // push 3 onto myvec

    // debug print myvec.
}
```

## Exercise 3b Box

```
fn main() {
    // Construct a box containing an array of 3 integers.
    // let mybox = ...;

    // Print the box.
    // println!(...)
}
```

### bonus questions

How many bytes does the box take:

* On the stack.
* On the heap.

What happens when we clone a Box?

## Exercise 3c Rc

`Rc` is a shared pointer for single threaded
code.

```

use std::rc::Rc;

fn main() {
    // Construct a shared pointer to an array of 3 integers.
    // let myrc = ...;

    // Print the rc.
    // println!(...)
}
```

### bonus questions

How many bytes does `myrc` take:

* On the stack.
* On the heap.

What happens when we clone a `Rc`?

When should we use an `Arc` instead of an `Rc`.

# Excercise 4a References



```
fn myfun(x: &i32) -> i32 {

}

fn main() {
    println!("myfun(&1)={}", myfun(&1));
}
```
