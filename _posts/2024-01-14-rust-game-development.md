---
layout: post
title:  "Game development in Rust"
date:   2024-01-14 14:12:06 +0000
categories: rust
---
C++ has long been the language of choice in professional game
engine development.

This was not always the case as C and assembler were dominant for
some time before C++ with games like Quake being the last vestiges
of this.

Of course, some games are written in interpreted languages, originally
BASIC and more recently JavaScript and some game engines such as Unity
use languages like C# for scripting. C# is middle ground between
script languages and C-like languages using the "everything's a pointer"
model and garbage collection. It is easier to learn than C++ and less
likely to experience undefined behaviour leading to crashes.

It should be noted that the runtimes of Unity and other game engines
that use C# are actually written in C++.

So why C++ and more recently, why Rust?

Andy Thomason has worked in the game industry since the 1970's developing
Namco console games and AI Chess players in Z80 assembler as a teenager.

He has worked for Sony twice (Psygnosis and SN Systems) doing research
in game technology such as the PS3 and Vita compilers.

## The C/C++ programming model.

C++ is based on C and in fact the original C++ compiler, CFront transcoded
C++ into C. Like many languages before, C uses as "Stack and Heap" model
to handle dynamically created objects.

If we write a C function with a variable

```C
void my_function() {
    int x = 1;
    printf("%d", x);
}
```
then the variable `x` is stored in a *stack* frame which is reserved
when we call `my_function` and removed when we return from the function.

We can also allocate objects that live longer using `malloc` to get a
pointer to the *heap* so that when we return from a function, we still
have our object.

This all looks like this:

```
+---------------+ top
+ Stack         +
+---------------+ SP
+               +
+ unallocated   +
+               +
+---------------+ BRK
+ Heap          +
+---------------+ bottom

```

When you call a function, the stack pointer, SP goes down
creating more space, when you return, SP goes up freeing the space.

The heap, by comparison, grows up from the bottom, but never shrinks.
Instead we divide the heap into *chunks* which are allocated by `malloc`
and freed up by `free`.

Thus the *lifetime* of objects on the stack is limited to the call
being made, but the *lifetime* of an object allocated on the heap
can be much longer.

## Advantages or C and C++

Engines written in C++ are much faster than those written in Java, GO
and C# because you get much closer to the machine. You can go a lot faster
still if you write in assembler, but those skills are in decline.

Writing in C++ does not make the code go faster on its own, but it gives
you a larger toolbox to work with and lets you talk to the hardware
more directly. On game consoles, for example, the C++ code is used
to write to hardware registers directly, bypassing bulky APIs.

C++ also lets you use multithreaded code and most modern C++ game
engines let you create huge number of tasks and events which
will be handled during the frame or over the course of many frames.

## Problems with C and C++

If an untrained driver sits in a formula one car and tries to drive
it, they will likely crash immediately and it is the same for
C and C++ - the program will crash.

Consider the following code:

```C
int *fred() {
    int x = 0;
    return &x;
}
```

This function returns the address of the variable `x`, but after
returning from this function, x is no longer there and writing
to it will likely cause a crash. This is a *dangling pointer*.

Finding this kind of fault is very hard in C++ and this puts a lot
of people off writing games in C++.

Another problem with multi-threaded code is the *race condition*.

Consider this code:

```C
   // Thread 1
   x = 1;
   y = 2;

   // Thread 2
   x = 3;
   y = 4;

   // Thread 3
   X = x;
   Y = y;
```

What is the value of (X, Y)? This can be (1, 2) (3, 4)
(1, 4) (undefined, 4) and so on. Many many faults in game
engines exist because of this.

## Rust is the successor to C++

Rust was designed to get the benefits of C++ without the
pain of having to worry about race conditions and dangling
pointers.

It is a complete redesign of C++ with only the modern bits
and a safety-orientated checking system. It encourages
a common style of code generation trhouh warnings about
variable names and has extensive security checks to
avoid some of the nasty network attacks that can disable
games and steal user information.

Rust has two modes *safe* and *unsafe*. Most of the code
is written in *safe* mode which gives you guantees that
avoid the problems of C++ but some code is *unsafe*
such as interactions with hardware.

For example, the dangling pointer example we give
is not possible in Rust:

```Rust
fn my_function() -> &i32 {
    let x = 1;
    &x
}
```

This will generate an error.

Likewise with the race condition example, passing writable
references to variables to other threads is not allowed
in safe Rust, so you don't have to worry about it.

## So why not just stick with C#?

C# also gives these guarantees and indeed if you are writing
small games with low performance requirements, then C#
may be exactly what you are looking for.

But for large games, such as the Disney Engine which is over 5M
lines of code, using C# is just not going to be possible
and if you want to create effects that Unity is not pre-wired
to support, then good luck.

Rust makes it much easier to write large, multi-threaded games,
do networking, build servers to host thousands of players
and many more things.

Rust has a *huge* collection of libraries which you can use
by adding a single line to the manifest file like the NPM
JavaScript manager. The *Cargo* build tool will download
source code of any of the hundreds of thousands of libraries
available and compile it on the spot.

In many ways, it is the ease of using libraries that makes
Rust number one choice in new technologies such as Block Chain
and Fintech.

## Who uses Rust in the game industry?

Some studios, like Embark in Sweden, have adopted Rust
and are pushing the ecosystem. We are on the verge
of seing a new generation of Rust game engines
become stable enough for large scale development.

See [Are We game Yet?](https://arewegameyet.rs/)
for a huge list of game engines and libraries.

For example libraries for:

* Windowing
* Audio
* Shaders
* 3D rendering
* Text rendering
* AI
* ECS (Enitity-component-system model)
* VR
* 3D format loaders
* Maths
* Mesh tools.

[Ecosystem](https://arewegameyet.rs/#ecosystem)

etc.

As well as a stack of fully formed game engines:

* Bevy
* Fyrox
* Amethyst
* ggez
* macroquad
* Piston

I've been using Bevy engine to do shader experiments with molecular
modelling, for example. Bevy uses Webgpu to make games that
can run on desktops, phones, browsers and many more platforms.

Bevy has VR support, networking and many more things but it
is still and "expert" level tool, it doesn't have the easy
gui that Unity has but suits my way of working.

[Bevy Game Engine](https://bevyengine.org/)

Fyrox has a GUI-driven scene generator and is orientated at
scripting like Unity:

[Fyrox Game Engine](https://fyrox-book.github.io/beginning/scripting.html)

Amethyst is also quite programmer orientated

[Amethyst Game Engine](https://amethyst.rs/)

As is Piston:

[Piston game engine](https://www.piston.rs/)

## What needs to happen

Most Rust game engines are very much orientated towards
programmers. For exmple, bulding a large open world survival
strategy game like Factorio in Bevy would be quite easy,
but would require some programming skill.

To become more mainstream, these game engines need to
develop GUI interfaces to allow non-programmers to build
games. Some have started in that direction, but we will
see technically orientated games long before we see
artist-lead FPS games, for example.

Still, if it were a choice of starting a new game
engine in C++ or in Rust, the smart money would go
for Rust as it is hugely popular and much easier
to build large projects without breaking the bank.

If your were to start learning a low level language
now, Rust would be the choice, especially as most
Rust jobs are work-from-home with Europe developing
as a centre for Rust digital nomads.

As a lifestyle, the open source world of Rust is much
preferable to being stuck in a room of hundreds of C++
programmers on an industrial estate in the middle of nowhere,
not to mention any game studios in particular!

