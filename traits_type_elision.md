---
title: Advanced Rust
sub_title: Traits, closures and type elision
author: Willem Vanhulle
options:
  end_slide_shorthand: true
---

![otiv](otiv.jpg)

---

## Contents

Mix of information and quizes (from Tolnay) about:

1. Traits
2. Trait bounds
3. Supertraits
4. Trait objects
5. Traits and closures
6. Type elision (`impl Trait`)


Who is already familiar with these topics?

---

## What is a trait?

A set of common behaviours of concrete types.

A simple example of a trait:

```rust
trait Animal {
    fn new(name: &'static str) -> Self;

    fn name(&self) -> &'static str;
    fn noise(&self) -> &'static str;

    fn talk(&self) {
        println!("{} says {}", self.name(), self.noise());
    }
}
```

Features:

- The struct implemented corresponds to the argument `self`.
- You can add default function implementations.

---

### How do you use traits?

Things we might like about traits:

- **Minimal**: just the methods you need. (You can use super-traits to extend.)
- **Composable**: not mutually exclusive with other traits. (You can use combined traits.)
- What else do you like about traits ...?

One of the inspirations behind the trait system of Rust:

> "Traits: Composable Units of Behaviour", Sharli (2003)

---

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

We implement a trait for a very reference type.
```rust 
trait Trait: Sized { 
    fn is_reference(self) -> bool;
}

impl<'a, T> Trait for &'a T {
    fn is_reference(self) -> bool { true }
}
```

We call the method on `0` and `'?'`.

```rust
fn main() {
    match 0.is_reference() {
        true => print!("1"),
        false => print!("0"),
    }

    match '?'.is_reference() {
        true => print!("1"),
        false => {
            impl Trait for char {
                fn is_reference(self) -> bool {
                    false
                }
            }
            print!("0")
        }
    }
}
```

<!-- column: 1 -->

Notice we implement the same trait for `char`.

What is the output of this program?

<!-- pause -->


#### Short answer

```
10
```

<!-- pause -->


#### Long answer


1. `0.is_reference()`
   1. We look for a method called `is_reference` defined for integers 
   2. `Trait` is not implemented for an integer type
   3. Method resolution inserts an auto-ref `&0`.
   4. We find a method `is_reference` defined for references
   5. We get `true`
2. `'?'.is_reference()`
   1. We look for methods named `is_reference`.
   2. We have access to all the `impl` blocks.
   3. We find an `impl Trait for char`
   4. We return `false`.


Summary:
- method resolution performs **auto-referencing**.
- `impl` blocks are **visible everywhere**.


---


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

We create a trait implement a few **inherent methods** for `S`.

```rust
trait Trait {
    fn f(&self);
    fn g(&self);
}

struct S;

impl S {
    fn f(&self) {
        print!("1");
    }

    fn g(&mut self) {
        print!("1");
    }
}
```

<!-- column: 1 -->

We implement a trait for `S`.

```rust
impl Trait for S {
    fn f(&self) {
        print!("2");
    }

    fn g(&self) {
        print!("2");
    }
}

fn main() {
    S.f();
    S.g();
}
```



<!-- pause -->

#### Short answer

```
12
```

<!-- pause -->

#### Long answer


1. Inherent method takes priority (use `Trait::f(&S)`)
2. Auto-ref during method resolution always prefers making something into `&` over making it into `&mut`.

---


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

```rust
trait Trait {
    fn f(&self);
}

impl Trait for u32 {
    fn f(&self) {
        print!("1");
    }
}

impl<'a> Trait for &'a i32 {
    fn f(&self) {
        print!("2");
    }
}

fn main() {
    let x = &0;
    x.f();
}
```

<!-- pause -->

<!-- column: 1 -->

#### Short answer

```
1
```

<!-- pause -->

#### Long answer

1. We start with `let x = &0;` 
2. We call `(&0).f()` so we need to find type `T` such that `f(&self)` is defined.
3. The simplest possibility is  `T = u32`
4. The method call becomes  `<u32 as Trait>::f(x)`.

Key point: auto-referencing always uses the **minimal amount of references**.

[See](https://dtolnay.github.io/rust-quiz/15)

---

## Polymorphism

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


### Generic data types

Types that **depend on other types** `T`.

`T` is a **type parameter** or a **generic**.

Primitive:

- References: `&T`
- Slices: `&[T]`
- Tuples
- Structs
- Enums

Standard library data types

- Collections: `HashMap`
- Result
- Option 
- Future
  


<!-- column: 1 -->

### Polymorphic functions

Functions that take a type parameter.

- Can be free-standing (outside `impl` blocks)
- Inside an `impl` block

You can start with a free version and then an `impl` version.

```rust
fn sum_collections<T, U, V, I1, I2>(iter1: I1, iter2: I2) -> Vec<V>
where
    T: Add<U, Output = V>,   
    I1: Iterator<Item = T>,   
    I2: Iterator<Item = U>,    
{
    iter1.zip(iter2)           
        .map(|(x, y)| x + y)    
        .collect()            
}
```

<!-- reset_layout -->

Type parameters can be restricted by **trait bounds** in both cases. (see later)



---

### `impl` blocks for generic data types

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

We define a new generic data type:

```rust
struct Point<T, U> {
    x: T,
    y: U,
}
```

We have to declare type variables for all type arguments of the data type (`T` and `U`).

Each type variable has to appear in the generic data type.

```rust
impl<T, U> Point<T, U> {
    pub fn new(x: T, y: U) -> Self {
        Point { x, y }
    }

    pub fn x(&self) -> &T {
        &self.x
    }

    pub fn y(&self) -> &U {
        &self.y
    }
}
```
<!-- column: 1 -->

You can also decide to only implement for specific concrete type parameters:

```rust
impl Point<f64, f64> {
    pub fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

This particular `impl` block is visible **everywhere**. This is something new for Rust.

---


## Traits with type parameters

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Traits for a whole **class of structs**.
 
A lot of standard library traits are generic:

```rust
pub trait PartialEq<Rhs = Self> {
    fn eq(&self, other: &Rhs) -> bool;
    fn ne(&self, other: &Rhs) -> bool { !self.eq(other) }
}
```
Notice the default value for `Rhs`.

It's straightforward to implement for our `Point` and tuples.

```rust
impl PartialEq<(i32, i32)> for Point<i32, i32> {
    fn eq(&self, other: &(i32, i32)) -> bool {
        self.x == other.0 && self.y == other.1
    }
}

let p = Point { x: 1, y: 2 };
let result = p == (1, 2);  // true

```
<!-- column: 1 -->

Other examples:

```rust
trait From<T>: Sized {
    fn from(T) -> Self;
}
```

The fallible version:

```rust
trait TryFrom<T>: Sized {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

(Notice `type Error`: an **associated type**.)

```rust
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}
```

Commonly used with `AsRef<Path>`.


As with generics for structs, generic traits **reduce code duplication**.

---


### What are associated types?

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Who knows what an ssociated type is?

<!-- pause -->


The usage of an associated type moves the declaration of the type parameter **below the name of the trait**: 

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```

Notice that only one associated type has a bound `IntoIter`.

**Question**: When can a generic trait be converted to a trait with associated types?

<!-- pause -->

**Answer**: There should be a 1-to-1 map from the generic type to its generic type parameters.

Imagine the following generic data type:

```rust
struct Buffer<T> { data: Vec<T> }
```

**Question**: How can we implement `IntoIterator` for this?

<!-- pause -->

<!-- column: 1  -->

Specify `T` as the associated type `Item`. 

```rust
impl<T> IntoIterator for Buffer<T> {
    type Item = T; 
    type IntoIter = std::vec::IntoIter<T>; 

    fn into_iter(self) -> Self::IntoIter {
        self.data.into_iter()
    }
}
```
There is only one type of item for a given `Buffer<T>` and the trait `IntoIterator`, so we use an associated type.

Benefits:

- The associated types (or their bounds) do not have to be explicitly specified when you use a trait with associated types.
- You can use the names of the associated types instead of their order of definition when you specify them

---


## Generic traits with associated types


Some traits have generic type parameters **and** associated types.

```rust
pub trait Add<Rhs = Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```


Always put as many generic type parameters as possible in the associated type position.


Does anyone have examples of combinations?

---

## Trait bounds

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->



Every type parameter can be restricted by **trait bounds**.

### Inline

The simplest way to is to restrict type parameters inline:

```rust
struct Point<T: Add<Output = T>> { x: T, y: T, }
```

This specifies a bound on `T`: `T` should be add-able according to the trait `Add` with associated type equal to `T`.

### Where

When the trait bound becomes too long to fit on a line, you can also use `where` blocks:

```rust
struct Point<T> where T: Add<Output = T> { x: T, y: T, }
```


<!-- pause -->

You might think that the bounds of the `struct` definition are "remembered" and that you can just do


<!-- column: 1 -->


```rust
impl<T> Add for Point<T> {
    type Output = Self;

    fn add(self, other: Self) -> Self::Output {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
```

<!-- pause -->

But no, you need to specify the trait bounds **again** when implementing inside an `impl` block.

```rust
impl<T: Add<Output = T>> Add for Point<T> {
    type Output = Self;

    fn add(self, other: Self) -> Self::Output {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
```

_Notice we always have to provide the associated types in `impl` blocks._

So remember to re-apply all the bounds on `impl` blocks (and omit them on the struct?).

---

## Blanket implementations

Implementations of a trait on any type that satisfies the trait bounds are called **blanket implementations**.

In the previous example, we `impl` the trait `Add` on the condition that `T` implements `Add`.

```rust
impl<T: Add<Output = T>> Add for Point<T> {
    type Output = Self;

    fn add(self, other: Self) -> Self::Output {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
```

Remember that `impl` blocks are visible **everywhere**. 

To prevent inconsistencies, blanket implementations of **foreign traits cannot be applied to foreign data types**.

[See](https://doc.rust-lang.org/book/ch10-02-traits.html)

---

## Help, my trait bounds are not satisfied!


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


What should you do?

<!-- pause -->

Don't panic.

<!-- pause -->


### Mismatched call types

Are you applying a function to something unexpected?

1. Where is the function defined?
2. Is the function defined in an `impl` block?
3. What are the type parameters of the `impl` block? What are the trait bounds on the type parameters?
4. Are there extra generic type parameters for the method? What are their trait bounds?

### Mismatched field types

Are you getting data type errors?
1. Where is the field defined?
2. Does the field take generic type parameters?
3. Are you currently in an `impl` block with trait bounds?
4. Do the generic type parameters occur in a generic trait?
5. Does the order of the declaration of the type parameters in the struct match the order in the generic trait?

<!-- column: 1 -->

Things that may be useful:
- The order of type parameters is irrelevant at the start of `impl` blocks and in `where` clauses.
- In a bound like `A: B + C` (a **combined trait**), we have to check `A: B` **and** `A: C`. 

Type-checking is all about looking for expressions that are **mal-typed** or invalid, things that may not be constructed using the rules of the language.

The better your understanding is of the way **type inference** by the compiler works, the easier it will be.

- If you like a mathematical approach, you might want to look into **type theory** and start with **natural deducation**.
- For a more pragmatic approach: "Programming Language Pragmatics", Scott (2015)

---

## What are super-traits?

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Who has used super-traits?

<!-- pause -->

### Basics of super-traits

The simplest example of a super-trait:

```rust
trait Person {
    fn name(&self) -> String;
}

trait Student: Person {
    fn university(&self) -> String;
}

trait Programmer {
    fn fav_language(&self) -> String;
}
```

The trait `Student` is a **sub-trait** of `Person`.

The trait `Person` is a **super-trait** of `Student`.

We can mix super-traits with combined traits:

```rust
trait CompSciStudent: Programmer + Student {
    fn git_username(&self) -> String;
}
```

<!-- column: 1 -->

Here the trait `CompSciStudent` is a subtrait of both super-traits `Programmer` and `Student`. 

This means that super-traits allow **multiple inheritance**.

### `impl` blocks and super-traits

We can only define `git_username` inside an `impl` for the trait `CompSciStudent` if we already chose concrete functions for all the methods of both super-traits and put them in an `impl` block.

```rust
struct Someone;

impl Programmer for Someone {
    ...
}

impl Student for Someone {
    ...
}

impl CompSciStudent for Someone  {
    fn git_username(&self) -> String {
        ...
    };
}
```

---

## Super traits and method resolution

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

We define two traits. One is a super-trait of the other.

```rust
pub trait SuperTrait {
    fn method(&self) {
        println!("Super")
    }
}

pub trait SubTrait: SuperTrait {
    fn method(&self) {
        println!("Sub")
    }
}
```

We `impl` both traits for `char` and use the default implementations.
```rust
impl SuperTrait for char { }
impl SubTrait for char { }
```

<!-- column: 1 -->


```rust
fn main() {
    '?'.method()
}
```

**Question**: what will this program output?

<!-- pause -->

**Answer**: Rust does not allow ambiguous method names


```txt
   |
24 |     '?'.method()
   |         ^^^^^^ multiple `method` found
```

Explicit qualification of the trait is necessary.

Overriding methods won't work. This rule is to keep code predictable. Traits can only **extend supertraits** with methods that are not yet in the super-trait.

---

## How to organize super-traits?

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Question**: How do you use super-traits?

<!-- pause -->

### Repeated trait bounds

The first place were super-traits can be useful is when we have different `impl` blocks that have the same set of trait bounds in `where` blocks.

1. Analyze what these repeated trait bounds actually mean.
2. Decide whether there is a "simple" and "extended" set of bounds.
3. Pick a name for a sub-trait (with the simple bounds)
4. Put most of the repeated bounds in the sub-trait.
5. Pick a name for a super-trait (with the extended bounds)
6. Move as many bounds one level higher-up to the super-trait.
7. Relax unnecessary bounds on the sub-trait and super-trait and move them to relevant methods.


<!-- column: 1 -->

**Question**: Can we place extra trait bounds on associated types in super-traits?

<!-- pause -->

**Answer**: Yes!

For example, if we have a super-trait defined as 

```rust
trait SuperTrait {
    type State;
    type SuperState;

    ....
}
```

Then we can specify a sub-trait that extends (implements) all `SuperTrait` for which `SuperTrat::State: Sync`.

```rust
trait SubTrait: SuperTrait<State: Sync> {}
```

Note that it is not necessary to specify bounds on **all the associated types**.

---

## Weird stuff

### Introducing new associated types

Let assume we still have the super-trait

```rust
trait SuperTrait {
    type State;
    type SuperState;
}
```

We can introduce a new associated type in `SubTrait`. Then you can refer to this new associated type in the `SuperTrait`.

```rust
trait SubTrait: SuperTrait<State=Self::SubState> {
    type SubState
}
```

Note that we don't have to specify all the associated types of the super-trait. 

### Invalid bounds

However, this won't work:

```rust
trait SuperTrait {
    type State;
}

trait SubTrait: SuperTrait<State: Self::SubState> {
    type SubState;
}
```

You cannot use a new associated type as a trait bound for an associated type in a super trait.


---

## What are some common trait bounds?

<!-- pause -->

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

### Send

All types that are safe to be moved between threads are `Send`. They are also called **thread safe**.

What does `Send` for a type mean in practice?
- It's ownership can be transferred to other thread
- The other thread becomes responsible for dropping object

Also applies to types sent to async tasks since they might have to go to **worker threads**.

Structs without references are `Send`.

**Not send** because of possibility of data races: `*mut T`

<!-- pause -->

<!-- column: 1 -->

### Sync

A value of type `T` is `Sync` **if and only if** the an immutable reference to `T` (`&T`) is `Send`.

In practice this means that it is safe to **access immutably from several threads** in parallel.

How can we derive if something is `Sync`? There are some rules:

- References are in one-to-one corresponds with references that point to references (`&T <=> &&T`). This implies that `T : Sync <=> &T : Sync`.
- A consequence of this is that if `T: !Sync` then `&T: !Sync`

The most common example of a `Sync` type is a shared immutable reference `Arc` which can be shared accross threads.

The single-threaded `Rc` is not `Sync` since it doeesn't use atomic variables underneath, but has lower overhead.

<!-- reset_layout -->

Automatically implemented for a given type `T` by the compiler if all the components of `T` are `Sync` or `Send`. This turns them into **auto-traits**.

**Question**: What are some other auto-traits?

<!-- pause -->

**Answer**: For example `Sized` (size determined at compile time), `Unpin` (can be moved out of a `Pin`).

---

### Inheritance of `Send` and `Sync`

**Question**: in which cases does `Arc<T>` implement `Send`?

<!-- pause -->

**Answer**: `T` must implement `Send`.

Why?

<!-- pause -->

`Arc<T>` may have to move `T` to another thread if the second-last handle to the reference counted variable is destroyed and the last handle is situated on another thread.

<!-- pause -->


**Question**: in which cases does `Arc<T>` implement `Sync`?
<!-- pause -->

**Answer**: `T` must implement `Sync` because `Arc<T>` gives us a `&T` and `&T` if and only if `T` is `Sync`.


To check whether a data type `T` is `Sync` or `Send` you have to start at the core and **extend your search outwards** for fields that are not `Send` or `Sync`.

---

## Combinations of `Sync` and `Send`

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

### `Send + !Sync`: 

Types that may be accessed from any thread but only one at a time

`Cell` and `RefCell` are  both not `Sync` because they have **interior mutability**. This means you can modify the value inside with only an immutable reference. This would be a data race if you could do it from several threads in parallel, so it is not `Sync`. 

They are `Send` if their inner type is `Send`.

### `!Send + !Sync` 

Types that manipulate shared handles to `!Sync` types

`Rc<T>`

- **not** `Send`: since you would be able to clone and then send it to a different thread. This would mean two different `Rc<T>` pointing to `T` from two different threads. `Rc<T>` is not built for that.
- **not** `Sync`: because `Rc<T>` is not `Send`, so `&T` is not `Send` and we know from before that `T : Sync <=> &T : Sync`.

Raw pointers (I don't know anything about this.)

<!-- column: 1 -->

### `!Send + Sync`:

Rare combination: may be accessed immutably from any thread and multiple in parallel

Example: `MutexGuard` of a `T`
- cannot be dropped in a different thread to prevent data races so it is not `Send`.
- If `T` is `Sync` then it follows from `T : Sync <=> &T : Sync` and from the fact that we can obtain a reference `&T` from `MutexGuard<T>`  that a `MutexGuard<T>` is `Sync`.


[See](https://users.rust-lang.org/t/example-of-a-type-that-is-not-send/59835/3)

<!-- reset_layout -->


[See](https://stackoverflow.com/questions/68704717/is-the-sync-trait-a-strict-subset-of-the-send-trait-what-implements-sync-withou)

---

## What about lifetime bounds?

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

We have seen trait bounds on type parameters.

It is also possible to have **lifetime bounds**.

### The basics

Life-times are used for static analysis.

You might already know: A reference `&'a T` has a life-time `'a` if it will be valid for the duration of `'a`. 

See [nomicon](https://doc.rust-lang.org/nomicon/lifetimes.html)

### Lifetime bounds on type parameters

The statement `T: 'a` means that all references in `T` are valid for `'a`. So we know that `&'a T => T: 'a`.

**Question**: For which lifetimes `'a` can you move a `T: 'a`?

<!-- pause -->

**Answer**: The bound is just a constraint on the life-times inside `T`, so you just have to make sure the life-time of references inside `T` are guaranteed.

A special kind of references are static references such as `&'static T`.  are references that remain **valid until the end** of the program.


<!-- column: 1 -->

### The special life-time `'static`

If `T` does not contain any non-`'static` references, then `T` receives a default lifetime bound `T: 'static`. `'static` is a reserved name for a lifetime. 


There are two possibilities:
- Either `T` does not contain any references and is an **owned variable**.
- `T` only contains references that remain valid for the rest of the program.

Where will you encounter `T: 'static`?

<!-- pause -->

- Some container types assume that `T: 'static` such as `Box<T>`.
- Types that are sent between threads or tasks have to be `T: 'static`.

---

## Life-times in traits

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

You can declare lifetime parameters for traits.

Most commonly for `struct`s that take lifetime parameters:

```rust
trait BorrowedData<'a> {
    fn get_data(&self) -> &'a str;
}
```

Let's say we have some struct that contains a reference:

```rust
struct DataHolder<'b> {
    data: &'b str,
}
```

We can implement the trait for the struct?:

```rust
impl<'a> BorrowedData<'a> for DataHolder<'a> {
    fn get_data(&self) -> &'a str {
        self.data
    }
}
```

<!-- column: 1 -->

### Associated types in super-traits

A trait may specify an associated type that depends on a life-time.

```rust
trait SuperTrait {
    type State<'a>;
}
```

Every `impl` block will need to provided a type parametrised by life-time `'a`.


When you want to add bounds you might try:

```rust
trait SubTrait: SuperTrait<State: Sync> { }
```

But this won't work. You need to declare a lifetime:

```rust
trait SubTrait: for<'a> SuperTrait<State<'a>: Sync> { }
``` 
(Notice how the `for<'a>` appears in front of the supertrait.)

---

## Function types

Things that **produce values**.

### Traditional function types

Rust has a few function-like things:
- Function items
- Function pointers
- Closures

**Question**: Which other things produce values?

<!-- pause -->


### Extended function types

To be complete, there are also things with `call`-like functionality such as
- iterators
- co-routines
- futures
- async iterators

You might be interested in effects: ways to model styles of computation. An effect is an additional “kind” in the formal, type system sense of the word, like a life-time.

See the programming languages: 

- [Koka](https://koka-lang.github.io/koka/doc/index.html)
- [Effekt](https://effekt-lang.org/)

---

## Function traits

Let's focus on Rust.

Function items, pointers and closures can be categorized **using the trait system** of Rust.

Function items and function pointers satisfy all these traits, but closures form a ladder:

1. `Fn`: closures that do not capture mutable references and may be called infinitely
2. `FnMut`: closures that may reference mutable variables
3. `FnOnce`: closures that can only be called once and may move or drop variables

The relationship between them is `Fn => FnMut => FnOnce`.

---

## Function items

Named function declarations that implement all function traits `Fn, FnMut, FnOnce`. 

May be on top-level or nested as **helper functions**.

```rust
fn foo<T>() { }
```

Or they an appear as methods in traits.


```rust
trait MyTrait {
    fn do_something(&self);
}

struct MyStruct;

impl MyTrait for MyStruct {
    fn do_something(&self) {
        println!("Doing something!");
    }
}
```

In this position they may take the special parameter `self`.



---

## Lifetime elision in free functions

When function item arguments or return types have reference types, **lifetime annotations have to be added**.

But in some cases they can be omitted. The rules that describe this are called **lifetime elisision rules**.

If there is exactly **one input lifetime position** (elided or not), that lifetime is assigned to all elided output lifetimes.


```rust
fn print(s: &str);                                      // elided
fn print<'a>(s: &'a str);                               // expanded

fn debug(lvl: usize, s: &str);                          // elided
fn debug<'a>(lvl: usize, s: &'a str);                   // expanded

fn substr(s: &str, until: usize) -> &str;               // elided
fn substr<'a>(s: &'a str, until: usize) -> &'a str;     // expanded

fn get_str() -> &str;                                   // ILLEGAL

fn frob(s: &str, t: &str) -> &str;                      // ILLEGAL
```

---

## Lifetime elision in methods

If there are multiple input lifetime positions, but one of them is &self or &mut self, the lifetime of self is assigned to all elided output lifetimes.


```rust
fn get_mut(&mut self) -> &mut T;                        // elided
fn get_mut<'a>(&'a mut self) -> &'a mut T;              // expanded

fn args<T: ToCStr>(&mut self, args: &[T]) -> &mut Command                  // elided
fn args<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // expanded
```


---

## Implementation function items

They become part of immutable code in the compiled binary.
- can easily be cloned or copied  
- can be optimized away `=>` no pointer needed `=>` zero-sized


The signature of function items:
- has to be declared in advance.
- afterwards, you can't really refer to it. (I do not not know the details.)
  
The compiler will reference to them using `fn(u32) -> i32 {fn_name}`.

Every distinct instantiation of a generic function item with different type parameters has its **own unique type** because the type itself **encodes the information of which body will be called**.

```rust
let x = &mut foo::<i32>;
*x = foo::<u32>; //~ ERROR mismatched types
```

**Question**: What is the solution?

<!-- pause -->

**Answer**: Cast them as function pointers (see in a few slides).

Recursion in function items is possible but discouraged.

---

## Closures

---

## Closures


### What are closures?

|                        | **Function items** | **Closures** |
|------------------------|----------------|----------|
| _Anonymous_ | No | Yes |
| _Capture_                | No             | Optional |
| _Recursive_              | Optional       | No       |
| _Signature_              | Required       | Optional |
| _`Fn` and `FnMut` trait_ | Yes            | Optional |
| _`FnOnce` trait_         | Yes            | Yes      |

Because they can be anonymous closures are useful for **functional programming**.

Functional programming languages require functions as **first-class citizens**.

We have to be able to pass them around anonymously, which is why they are also called **anonymous functions**.

In lambda-calculus closures are written using a lambda, they are also called **lambda-functions**

For example:

```
(λx. x + 1) 5 
```

A foundation for formal verification of programming languages. 

See [Plato Stanford](https://plato.stanford.edu/entries/lambda-calculus/)

---

## Closures

### Async closures

In this presentation

On Rust stable, closures cannot be `async`. You need to do a heap allocation of a **trait object** (see further): `Pin<Box<dyn Future>>`.

On Rust nightly, closures can be `async` with `#![feature(async_closure)]`.

```rust
// Instead of writing:
fn doesnt_exactly_take_an_async_closure<F, Fut>(callback: F)
where
    F: FnOnce() -> Fut,
    Fut: Future<Output = String>
{ todo!() }

// Write this:
fn takes_an_async_closure<F: async FnOnce() -> String>(callback: F) { todo!() }
// Or this:
fn takes_an_async_closure<F: AsyncFnOnce() -> String>(callback: F) 
```


[See](https://blog.rust-lang.org/inside-rust/2024/08/09/async-closures-call-for-testing.html)

---

## Closures and lifetimes

Will the following code compile?

```rust
fn main() {
    let closure = |x: &i32| x;
}
```

<!-- pause -->

**Answer**: No, because lifetime elision rules are different for closures compared to function items.


```
  |
2 |     let closure = |x: &i32| x;
  |                       -   - ^ returning this value requires that `'1` must outlive `'2`
  |                       |   |
  |                       |   return type of closure is &'2 i32
  |                       let's call the lifetime of this reference `'1`
```

The closure definition can be expanded to:

```rust 
fn main() {
    let closure = for<'a, 'b> |x: &'a i32| -> &'b i32 { x };
}
```

Two distinct lifetimes are created for the input and output type. For normal functions, only one lifetime is created.


See [Pretzelhammer](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#10-closures-follow-the-same-lifetime-elision-rules-as-functions)

---

## Closure implementations

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

The implementation of closures is taken for granted in most languages. Rust is more low-level and it is important to know how they work under the hood.

The **body block of the closure is analyzed**: 
1. variables in the body that refer to variables in the surrounding scope are marked as captured
2. struct generated at compile time with as fields the references to the captured variables, it serves as the environment for the closure body
  - the struct is invisible and out of reach for the programmer in normal Rust code
  
### Types of closures

- the direct type of a closure is implicit and cannot be used directly
- so closures are called **unnameable types** (also called **Voldemort types**)

[See](https://huonw.github.io/blog/2015/05/finding-closure-in-rust/)

<!-- column: 1 -->

### Rough model

A simple closure may be roughly modeled as follows:

```rust
let mut v = vec![];
let closure = || v.push(1);

struct Environment<'v> {
    v: &'v mut Vec<i32>
}

impl<'v> Fn() for Environment<'v> {
    fn call(&self) {
        self.v.push(1) // error: cannot borrow data mutably
    }
}
let closure = Environment { v: &mut v }
```



### Marker traits on closures:

- just like structs, closures may implement `Copy`, `Clone`,  `Send` and `Sync`
- they only implement marker traits when captured content does.


---
## Variants of closures


### `FnOnce`

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


The least a closure should be able to do is be called once.

How is a closure that can be called once implemented?

Let's modify the rough model a bit. 

```rust
let v = vec![];
let closure = || v.push(1);

struct Environment {
    v: Vec<i32>
}

impl FnOnce() for Environment {
    fn call(self) {
        self.v.push(1)
    }
}
let closure = Environment { v }
```

Notice that the environment takes ownership of the local variable `v`. 

<!-- column: 1 -->

Notice that the signature of `call` is written as `call(self)` which means that it's type is `Environment -> ()`.

The body of the call function:
- may mutate captured variables or references
- May move variables that are moved in, back out
  - consume captured variables and drop them
  - apply functions to them and call by value, not by reference

---

### Move

In practice, closures cannot just consume their environment. 

You need to explicitly mark closures as **consuming** their environment with a keyword `move`.
- takes ownership of variables in the surrounding scope
- inside the closure the captured and moved variables are used by reference


```rust
let data = vec![1, 2, 3];
let closure = move || println!("captured {data:?} by value");
```


**Question**: is every `move` closure `FnOnce`.

<!-- pause -->

**Answer**: No, the captured references are moved inside, but the closure can be called multiple times since the body uses the captured variables by reference.

---

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

```rust
trait Trait {
    fn f(&self);
}

impl<F: FnOnce() -> bool> Trait for F {
    fn f(&self) {
        print!("1");
    }
}

impl Trait for () {
    fn f(&self) {
        print!("2");
    }
}

fn main() {
    let x = || { (return) || true; };
    x().f();
}
```


<!-- column: 1 -->

<!-- pause -->

#### Short answer

This will output 2. Why?

<!-- pause -->

#### Long answer

1. We define a closure x `|| { (return) || true; }`
2. `(return)` is of type `!` because it never completes
3. `(return) || true` is of type `! || true` which evaluates to `bool || true`
4. `bool || true;` evaluates to `()`
5. `f` is implemented for `()` to output `2`.

```rust
fn main() {
    let x = || { return || true; };
    x().f();
}
```
Will output 1 since a call to x returns another closure that returns a bool.


```rust
struct Environment<'v> {
    v: &'v mut Vec<i32>
}

impl<'v> Fn() for Environment<'v> {
    fn call(&mut self) {
        self.v.push(1)
    }
}
```

### Closures that are `FnMut`

The signature is `call(&mut self)`

- can have
    - mutable references to its containing scope (but importantly, does **not need to**)
    - immutable references to its containing scope
- cannot consume or move out captured variables
- can be called more than once, but only once at a time, so it must implement `FnOnce`

```rust
let mut x = 5;
{
    let mut square_x = || x *= x;
    square_x();
}
assert_eq!(x, 25);
```

---

### `Fn` function trait

We modify the signature of the `call` method slightly.

```rust
struct Environment<'v> {
    v: &'v mut Vec<i32>
}

impl<'v> Fn() for Environment<'v> {
    fn call(&self) {
        self.v.push(1) // error: cannot borrow data mutably
    }
}
```

The signature is `call(&self)`.

Properties of `Fn` closures
- the body may only have
    - immutable references to its containing scope
    - values that were moved from the containing scope (and only use them by reference afterwards)
- can be called from anywhere, multiple times
- Must implement `FnMut`



---
## Function pointers

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Not necessarily named functions that implement `Fn, FnMut, FnOnce`

A function pointer is a type declared with `fn() -> ()`. 

### Construction

The first way to create a function pointer is by **casting a function item**:

```rust
fn foo<T>() { }
let x: fn() -> () = foo::<i32> as fn();

```

The second way is by casting a non-capturing closure.

```rust
let x: fn() -> () = || {};
```

<!-- column: 1 -->

Important: 
- it is **not a trait** `=>` cannot be used as a trait bound.
- cannot capture from the environment like a closure


Implementation: it has the size of a pointer, so it has to be dereferenced:
- might be slower than calling a function item or closure directly
- faster then `dyn Fn`

---

### Guess the size (part 1)


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

Predict the output of

```rust
fn my_function(x: i32) -> i32 {
    x + 1
}

fn main() {
    let f = my_function;
    println!("Size of f: {}", std::mem::size_of_val(&f));
}
```

<!-- pause -->


#### Answer

```
0
```

Why?

<!-- pause -->

Function items **do not have a run-time cost**, so they do **not require memory allocation**.


<!-- column: 1 -->


### Guess (part 2)


What is the output of: 

```rust
fn my_function(x: i32) -> i32 {
    x + 1
}

fn main() {
    let f: fn(i32) -> i32 = my_function; 
    println!("Size of f: {}", std::mem::size_of_val(&f)); 
}
```

<!-- pause -->

### Answer

```
8 (64-bit)
4 (32-bit)
```



Function pointers are stored in a pointer-like form that can be used at run-time.

---

### Lifetime notation of function pointers

```rust
struct S {
    function_pointer: for<'a> fn(arg: &'a i32) -> &'a i32
}
```

---

### What to do with function pointers?

Can be passed as argument to other functions or returned.

Can be an **associated item** of a trait

```rust
pub trait IntoStateMachine
{
    ...
    const ON_TRANSITION: fn(&mut Self, &Self::State, &Self::State) = |_, _, _| {};
}
```

Beware that you cannot use `impl` in the signature. So async functions can only be written as function pointers if you transform them in normal function pointers that allocate on the heap.

Particularly useful when working with languages that don't support closures like C.


---

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

### Question

What is the output of?

```rust
fn d<T>(_f: T) {
    match std::mem::size_of::<T>() {
        0 => print!("0"),
        1 => print!("1"),
        _ => print!("2"),
    }
}

fn a<T>(f: fn(T)) {
    d(f);
}

fn main() {
    a(a::<u8>);
    d(a::<u8>);
}
```

<!-- column: 1 -->

<!-- pause -->

#### Short answer

```
20
```

<!-- pause -->

#### Long answer

1. The first call in main coerces `a::<u8>` from a function to a function pointer (`fn(fn(u8)) {a::<u8>}` to `fn(fn(u8))`) prior to calling d, so its size would be 8 on a system with 64-bit function pointers
2. The second call in main does not involve function pointers; `d` is directly called with `T` being the inexpressible type of `a::<u8>`, which is zero-sized.


[See](https://dtolnay.github.io/rust-quiz/34)

---

### Function pointer fields



<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

```rust
struct S {
    f: fn(),
}

impl S {
    fn f(&self) {
        print!("1");
    }
}

fn main() {
    let print2 = || print!("2");
    S { f: print2 }.f();
}
```

<!-- column: 1 -->

<!-- pause -->

#### Short answer

```
1
```

<!-- pause -->


#### Long answer

A call that looks like `.f()` always resolves to a method, in this case the inherent method `S::f`. To call the function pointer stored in field `f`, we would need to write parentheses around the field access: 

```rust
fn main() {
    let print2 = || print!("2");
    (S { f: print2 }.f)();
}
```

(See Tolnay)


---
### Ranges and FnOnce



<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


#### Question
```rust
use std::ops::RangeFull;

trait Trait {
    fn method(&self) -> fn();
}

impl Trait for RangeFull {
    fn method(&self) -> fn() {
        print!("1");
        || print!("3")
    }
}

impl<F: FnOnce() -> T, T> Trait for F {
    fn method(&self) -> fn() {
        print!("2");
        || print!("4")
    }
}
```

<!-- column: 1 -->

What does this print?

```rust
fn main() {
    (|| .. .method())();
}
```

<!-- pause --> 

#### Short answer

In this case main would print 24.

#### Long answer

This code is parsed as 

1. `(|| ..).method()`
2. this an invocation of the impl of `Trait` on `FnOnce() -> T` where `T` is inferred to be `RangeFull`. 

[See](https://dtolnay.github.io/rust-quiz/33)

---

## Iterators



<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


```rust
struct Fibonacci {
    curr: u32,
    next: u32,
}

impl Iterator for Fibonacci {
    type Item = u32;
    fn next(&mut self) -> Option<Self::Item> {
        let current = self.curr;

        self.curr = self.next;
        self.next = current + self.next;
        Some(current)
    }
}
```

<!-- column: 1 -->

Return a Fibonacci sequence generator

```rust
fn fibonacci() -> Fibonacci {
    Fibonacci { curr: 0, next: 1 }
}

fn main() {
    // `0..3` is an `Iterator` that generates: 0, 1, and 2.
    let mut sequence = 0..3;

    println!("Four consecutive `next` calls on 0..3");
    println!("> {:?}", sequence.next());
    println!("> {:?}", sequence.next());
    println!("> {:?}", sequence.next());
    println!("> {:?}", sequence.next());
```

---

## Into iter vs. iter

Iterators have to be generated from iterable data structures. 



<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

### Creating iterators

There are multiple ways to create iterators.

**Question**: What is the difference between `into_iter` and `iter`?

<!-- pause -->

**Answer**
- The iterator returned by `into_iter` may yield any of `T`, `&T` or `&mut T`, depending on the context.
- The iterator returned by iter will yield `&T`.
<!-- pause -->

### For loops

**Question**: Does a for loop use `into_iter` or `iter`?

<!-- pause -->

**Answer**: `into_iter`

<!-- column: 1 -->

A for loop desugars to:

```rust
let mut it = values.into_iter();
loop {
    match it.next() {
        Some(x) => println!("{}", x),
        None => break,
    }
}
```

[See](https://hermanradtke.com/2015/06/22/effectively-using-iterators-in-rust.html/)

---

### Lazy map

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


#### Question

```rust
fn main() {
    let input = vec![1, 2, 3];

    let parity = input
        .iter()
        .map(|x| {
            print!("{}", x);
            x % 2
        });

    for p in parity {
        print!("{}", p);
    }
}
```

<!-- column: 1 -->

<!-- pause -->
#### Short answer
```
112031
```

<!-- pause -->


#### Long answer

The closure provided as an argument to map is only invoked as values are consumed from the resulting iterator. The closure is not applied eagerly to the entire input stream up front.

[See](https://dtolnay.github.io/rust-quiz/26)

---

## Dealing with complex types

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Sometimes
- we don't know the type
  - the particular instance of a trait we exactly need as input or output for a function signature
  - the actual type is hidden from the user. These types are called unnameable or Voldemort types
- we know the type, but the full type is too long to be readable
  - iterator implementors
  - future combinators (see next session)

**Question**: How to deal with such types?

<!-- pause -->

**Answer**: Use **type elision**
- For local variables 
  - use the wildcard `_` for generic parameters and let the compiler infer
- In type declarations for functions, traits or structs, 
  - the type cannot be inferred by the compiler
  - use opaque types

<!-- column: 1 -->
   
Benefits of opaque types
- hide specific types,
- cleaner API
- underlying concrete type can be an unnameable type
   
---

### Trait objects

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

When don't know the concrete type until runtime, we can replace the concrete type by a trait object and call methods through **dynamic dispatch** (marked with the `dyn` keyword).

```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

<!-- column: 1 -->


multiple concrete types to fill in for the trait object at runtime `=>` type elision


How is it implemented?
- a pointer to an object
- a pointer to a method table, called a **vtable** with function pointers for each method of the trait


Disadvantages
- Cannot be pushed on the stack directly, 
- has to be on the heap or behind a pointer on the stack
- Method call is determined at runtime, less optimizations

---

### Creating trait objects 

**Question**: Which of the following traits can be used to create a trait object?
- A trait with methods that have generic parameters
- A trait with a method that returns Self

<!-- pause -->

**Answer**: neither of them.


Traits that can be turned into a trait object are called **object safe**.

> You must be able to call methods of the trait the same way for any instance of the trait
> properties guarantee that, no matter what, 
> - the size and shape of the arguments and return value only depend on the bare trait — 
> - not on the instance (on Self) or any type arguments (which will have been "forgotten" by runtime)

[See](https://www.reddit.com/r/rust/comments/kw8p5v/what_does_it_mean_to_be_an_objectsafe_trait/)

---


### DSTs

Trait objects are part of the family of **dynamically sized types** (DST) 

- Their type is unknown until run-time, so their size is unknown until run-time.
- They cannot be allocated on the stack.

All generic type parameters should have a determined size at compile time by default.

This means that `GenericType<D>` is syntax sugar for `GenericType<D: Sized>` 

If you want to use a DST, you have to use `?Sized` as in `GenericType<D: ?Sized>`.
  
**Question**: What are fat pointers? Why are they called fat pointers?

<!-- pause -->

**Answer**: References to DSTs are called **fat pointers.** A fat pointer contains a pointer plus some information that makes the DST "complete":
- the length
- a pointer to the method table
  

[See](https://stackoverflow.com/questions/57754901/what-is-a-fat-pointer)


---

### Dynamic versus static dispatch

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

```rust
trait Base {
    fn method(&self) { print!("1"); }
}

trait Derived: Base {
    fn method(&self) { print!("2"); }
}

struct BothTraits;
impl Base for BothTraits {}
impl Derived for BothTraits {}

fn dynamic_dispatch(x: &dyn Base) {
    x.method();
}

fn static_dispatch<T: Base>(x: T) {
    x.method();
}
```


<!-- column: 1 -->

What is the output of

```rust
fn main() {
    dynamic_dispatch(&BothTraits);
    static_dispatch(BothTraits);
}
```

<!-- pause -->

11

---

#### Long answer 

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


- Dynamic dispatch:
  - convert the argument of type BothTraits to `dyn Base`
  - Method call can be expanded to `<dyn Base as Base>::method`.
  - Method from a table of function pointers contained within the trait object. 
  - print 1.
- Static dispatch: 
  - do type inference at compile time
  - look at the trait bounds of the generic parameter
  - determine the function to be called
  - expand the call as `<T as Base>::method` or `<BothTraits as Base>::method` 
  - concrete instantiation of the generic type parameter
  - print 1.

```rust
fn main() {
    dynamic_dispatch(&BothTraits);
    static_dispatch(BothTraits);
}
```

<!-- column: 1 -->


```rust
trait Base {
    fn method(&self) { print!("1"); }
}

trait Derived: Base {
    fn method(&self) { print!("2"); }
}

struct BothTraits;
impl Base for BothTraits {}
impl Derived for BothTraits {}

fn dynamic_dispatch(x: &dyn Base) {
    x.method();
}

fn static_dispatch<T: Base>(x: T) {
    x.method();
}
```
---

## Lifetimes for trait objects

a trait object's lifetime bound is inferred from context

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


### Box


```rust
trait Trait {}
```


```rust
type T1 = Box<dyn Trait>;
```
What is the lifetime bound on `dyn Trait`? 
<!-- pause -->
```rust
type T2 = Box<dyn Trait + 'static>;
```
`Box<T>` has no lifetime bound on `T`, so inferred as `'static`.

<!-- column: 1 -->

<!-- pause -->

### Impl blocks

```rust
impl dyn Trait {}
```
What is the lifetime bound on `dyn Trait`?

<!-- pause -->
```rust
impl dyn Trait + 'static {}
```

### References

```rust
type T3<'a> = &'a dyn Trait;
```
<!-- pause -->

`&'a T` requires `T: 'a`, so inferred as `'a`

```rust
type T4<'a> = &'a (dyn Trait + 'a);
```


---

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->



`Ref` wraps a borrowed reference to a value in a RefCell box.

```rust
use std::cell::Ref;

// elided
type T5<'a> = Ref<'a, dyn Trait>;
// expanded, Ref<'a, T> requires T: 'a, so inferred as 'a
type T6<'a> = Ref<'a, dyn Trait + 'a>;


```

<!-- column: 1 -->


```rust
trait GenericTrait<'a>: 'a {}

// elided
type T7<'a> = Box<dyn GenericTrait<'a>>;
// expanded
type T8<'a> = Box<dyn GenericTrait<'a> + 'a>;

// elided
impl<'a> dyn GenericTrait<'a> {}
// expanded
impl<'a> dyn GenericTrait<'a> + 'a {}
```

[See](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#6-boxed-trait-objects-dont-have-lifetimes)



---

### `impl Trait`

The other type of opaque types or type elision.

Syntax sugar for hardcoding a specific type that can be inferred by the compiler

Benefits:
- No extra heap allocation
- No dynamic dispatch overhead

Synonyms
- anonymous types

---

## Basic example

```rust
fn parse_csv_document<R: std::io::BufRead>(src: R) -> std::io::Result<Vec<Vec<String>>> {
    src.lines()
        .map(|line| {
            // For each line in the source
            line.map(|line| {
                // If the line was read successfully, process it, if not, return the error
                line.split(',') // Split the line separated by commas
                    .map(|entry| String::from(entry.trim())) // Remove leading and trailing whitespace
                    .collect() // Collect all strings in a row into a Vec<String>
            })
        })
        .collect() // Collect all lines into a Vec<Vec<String>>
}
```

---

## `impl Trait` in arguments

Since the generic type parameter `R` only appears once (in the **argument position** and **without multiple trait bounds**), there is no use for declaring it explicitly. We can replace it by an `impl Trait` notation (an anonymous function argument type):

```rust
fn parse_csv_document(src: impl std::io::BufRead) -> std::io::Result<Vec<Vec<String>>> {
    src.lines()
        .map(|line| {
            // For each line in the source
            line.map(|line| {
                // If the line was read successfully, process it, if not, return the error
                line.split(',') // Split the line separated by commas
                    .map(|entry| String::from(entry.trim())) // Remove leading and trailing whitespace
                    .collect() // Collect all strings in a row into a Vec<String>
            })
        })
        .collect() // Collect all lines into a Vec<Vec<String>>
}
```

[See](https://doc.rust-lang.org/rust-by-example/trait/impl_trait.html)

---

### `impl` in return type


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->
```rust
use std::iter;
use std::vec::IntoIter;

// This function combines two `Vec<i32>` and returns an iterator over it.
// Look how complicated its return type is!
fn combine_vecs_explicit_return_type(
    v: Vec<i32>,
    u: Vec<i32>,
) -> iter::Cycle<iter::Chain<IntoIter<i32>, IntoIter<i32>>> {
    v.into_iter().chain(u.into_iter()).cycle()
}
```

<!-- pause -->

```rust
// This is the exact same function, but its return type uses `impl Trait`.
// Look how much simpler it is!
fn combine_vecs(
    v: Vec<i32>,
    u: Vec<i32>,
) -> impl Iterator<Item=i32> {
    v.into_iter().chain(u.into_iter()).cycle()
}
```

<!-- column: 1 -->

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v2 = vec![4, 5];
    let mut v3 = combine_vecs(v1, v2);
    assert_eq!(Some(1), v3.next());
    assert_eq!(Some(2), v3.next());
    assert_eq!(Some(3), v3.next());
    assert_eq!(Some(4), v3.next());
    assert_eq!(Some(5), v3.next());
    println!("all done");
}
```
[See](https://doc.rust-lang.org/rust-by-example/trait/impl_trait.html)

---

#### `impl` everywhere

1. Signatures of methods in traits can use `impl Trait`, also known as "return position impl Trait" (RPIT) 
   - Implemented using a kind of associated types [See](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html)
2. Type aliases can use `impl Trait`: `type I = impl Iterator<Item = u32>`. 
3. Traits with associated types can use `impl Trait` as bounds for the associated type (in nightly Rust)
4. In local variable bindings `let x: impl Future = foo()` (planned) 


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = Widget>;
}

impl Container for MyContainer {
    fn items(&self) -> impl Iterator<Item = Widget> {
        self.items.iter().cloned()
    }
}
```

<!-- column: 1 -->

```rust
struct SubsetWrapper<'a> {
    everything: &'a HashMap<usize, i32>,
    subset_ids: &'a [usize],
}

impl<'a> IntoIterator for &SubsetWrapper<'a> {
    type Item = &'a i32;
    type IntoIter = impl Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter {
        self.subset_ids
            .iter()
            .map(|id| self.everything.get(id).unwrap())
    }
}
```

[See](https://rust-lang.github.io/impl-trait-initiative/)

---
## New lifetime capture rules


Rust 2021: the following compiles because `impl` does not capture `s`
```rust
fn indices<'s, T>(
    slice: &'s [T],
) -> impl Iterator<Item = usize> {
    0 .. slice.len()
}
```

<!-- pause -->

Rust 2024: `impl` will capture lifetime parameter `'s`.
```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let i = indices(&data);
    data.push(4); // <-- Error!
    i.next(); // <-- assumed to access `&data`
}
```

New default = the hidden types for a return-position impl Trait can use any generic parameter in scope.

---

## New syntax


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Introduction of **use bound** `impl Trait + use<'x, T>`: 
  - the hidden type is allowed to use `'x` and `T`
  - but no other generic parameters in scope

To exempt from lifetime parameter capture, use `use<>`

```rust
fn indices<'s, T>(
    slice: &'s [T],
) -> impl Iterator<Item = usize> + use<> {
    //                             -----
    //             Return type does not use `'s` or `T`
    0 .. slice.len()
}
```

<!-- column: 1 -->

Advantage: fine control over capturing of lifetimes in the arguments


[See](https://blog.rust-lang.org/2024/09/05/impl-trait-capture-rules.html)

---


### Summary

- Trait bounds
- Closures
- Dynamic dispatch
- Impl traits

---

### Questions?

This presentation was made using [`presenterm`](https://github.com/mfontanini/presenterm).