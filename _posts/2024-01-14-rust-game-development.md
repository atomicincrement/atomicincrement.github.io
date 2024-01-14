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

* Part 1 - The case for Rust in games.
* Part 2 - An example: breakout in Bevy

*Andy Thomason has worked in the game industry since the 1970's developing
Namco console games and AI Chess players in Z80 assembler as a teenager.*

*He has worked for Sony twice (Psygnosis and SN Systems) doing research
in game technology such as the PS3 and Vita compilers.*

# Part 1 - The case for Rust in games.

## The C/C++ programming model - Stack and Heap

C++ is based on C and in fact the original C++ compiler, CFront transcoded
C++ into C. C uses as "Stack and Heap" model to handle dynamically created objects.

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

In garbage collected languages like C#, objects are *only* allocated
on the heap which is much costlier than allocating on the stack.

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

# Part 2 - An example: breakout in Bevy

To illustrate what it is like to write a game in Rust, lets start
with one of the examples from the Bevy game engine:

[Breakout](https://github.com/bevyengine/bevy/blob/main/examples/games/breakout.rs)

Like in C and C++, the entry point to a Rust program is `main()`

```Rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .run();
}
```

If this is all we did, then we would get a blank window.

So what we need to do is add data and code to make breakout run.

This adds two data *resources*, a scoreboard which we will define
and a clear colour which is a system defined resource.
Resources are not *assets* and do not draw themselves. We need
*entities* for the rendering plugin to draw anything, for example.
Resources are just bits of data which we will use.

```Rust
        .insert_resource(Scoreboard { score: 0 })
        .insert_resource(ClearColor(BACKGROUND_COLOR))
```

Next we add an event, which we will use to signal collisions
between the ball and other entities.

```Rust
        .add_event::<CollisionEvent>()
```

And to make the game work, we have some systems which are
functions that get called to update things.

```Rust
        .add_systems(Startup, setup)
        .add_systems(
            FixedUpdate,
            (
                apply_velocity,
                move_paddle,
                check_for_collisions,
                play_collision_sound,
            ).chain(),
        )
        .add_systems(Update, (update_scoreboard, bevy::window::close_on_esc))
```

The `.chain()` makes these functions get run in sequence. Bevy
is a multi-threaded game engine and may run systems in any order
on different threads if needs be.

The first of these systems is `setup` which is called at the Start
of the game.

```Rust
// Add the game's entities to our world
fn setup(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    asset_server: Res<AssetServer>,
) {
    // ...
}
```

The parameters to setup can come in any order and
use Rust's flexible type system to make Assets and
other components acessible to the function.

The `commands` parameter is an interface that lets you
change the state of the game. For example:

```Rust
    commands.spawn(Camera2dBundle::default());
```

sets up a 2D camera for the game world.

```Rust
    // Sound
    let ball_collision_sound = asset_server.load("sounds/breakout_collision.ogg");
    commands.insert_resource(CollisionSound(ball_collision_sound));
```

adds a sound resource to the game.

```Rust
    commands.spawn((
        SpriteBundle {
            // ...
        },
        Paddle,
        Collider,
    ));
```

Adds a *bundle* of components to an entity (the paddle). A bundle is
an easy way of deploying a number of components at a time.

The component system is similar to Unity. Each object in the game
world has a number of components such as `Transform` and `Sprite` as
well as some user-defined components.

The `Transform` component, for example, specifies the location
of a sprite and the `Sprite` component describes the colour, image
and other properties.

Here `Paddle` and `Collider` are user defined components.

Likewise, we spawn entities such as the ball, the bricks, the walls
and so on.

### Making custom components

Making custom components is easy in Bevy. We use a `derive` macro
to generate extra code needed for the component. In these two cases
there is no extra data needed, so the structs don't need curly braces:

```Rust
#[derive(Component)]
struct Paddle;

#[derive(Component)]
struct Ball;
```

The types, however, are used to make a distinction between `Paddle`
and `Ball` and will be used to select the components when we run the systems.

### Moving the paddle

To move the paddle, we need a system which takes user input and
all the entities which have `Transform` and `Paddle` components like this:

```Rust
fn move_paddle(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut query: Query<&mut Transform, With<Paddle>>,
    time: Res<Time>,
) {
    let mut paddle_transform = query.single_mut();
}
```

there is only one paddle, so `query.single_mut();` will do
and also enable us to write to the transform (move the paddle).

By default in Rust, references like `&Transform` are read-only
and we need to use `&mut Transform` and `single_mut` to allow
us to change the transform.

The rest of the function reads the keyboard and moves the paddle.

### Checking for collisions

```Rust
fn check_for_collisions(
    mut commands: Commands,
    mut scoreboard: ResMut<Scoreboard>,
    mut ball_query: Query<(&mut Velocity, &Transform), With<Ball>>,
    collider_query: Query<(Entity, &Transform, Option<&Brick>), With<Collider>>,
    mut collision_events: EventWriter<CollisionEvent>,
) {
    // ...
}
```

This system has:

* An interface to change the engine state.
* A writeable `Scoreboard` resource.
* A query to find the Velocity and Transform of the ball.
* A query to find anything with a Transform and a collider, which may be a brick.
* An `EventWriter` to signal collisions to other systems.

We spin round checking the ball position against the bricks and walls, updating the scoreboard
and sending events if anything collides.

### Sounds

```Rust
fn play_collision_sound(
    mut commands: Commands,
    mut collision_events: EventReader<CollisionEvent>,
    sound: Res<CollisionSound>,
) {
    // ...
}
```

Here we receive collision events and convert them into
sounds.

We create the sound by *spawning* the collision `sound`
in a bundle. Yes! sounds are entities too.

## Multithreading

Because of the danger of race conditions, Bevy is careful
not to call two systems at the same time with a mutable
reference to the same component.

Bevy's use of `#[derive]` and the Rust type system
makes for a more C#-like development environment.

With very large games, with hundreds of thousands of entitites,
this will make a big difference.

# That's all folks

We talked a little about how Rust, a low level language,
makes it easier to write safe multi-threaded code, stealing
some thunder from C# and giving a significant performance
boost.

We showed you how easy it is to build games using the
Bevy ECS (Entity-component-system) model.

So happy Rusting, and if you get the opportunity,
try writing a game in Bevy. It may take a bit of
getting used to, but you are a champion!
