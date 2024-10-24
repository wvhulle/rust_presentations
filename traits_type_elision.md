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

- Traits
- Generics
- Super-traits
- Common trait bounds
- Life-time bounds
- Function traits
- Function items
- Closures
- Impl trait
- Trait objects
- Iterators
- Fallible computations


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

- The value for which we want to call a method is `self`. 
- The type of `self` is `Self`, but it can also be, for example, `Rc<Self>`.
- You can add default function implementations.

---

## Orphan rule

**Question**: What does **foreign** mean in the context of traits and types? 

<!-- pause -->

**Answer**: Things defined in other crates and in the standard library.

To prevent inconsistencies after imports, there is the **orphan rule**.

Orphan rules:
- foreign traits cannot be implemented for foreign data types
- you cannot write `impl` blocks (without trait) for foreign types 

**Question** Why? 

<!-- pause -->

**Answer**: `impl` blocks are visible everywhere. Implementing a trait for a struct in a crate, shouldn't be able to cause another dependent crate to stop compiling. This means there are restrictions on defining `impl` blocks.

---

## Type aliases


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Imagine you have some cool type called `Session`. 

For some reason, it cannot be `clone`.

But you want to share it.

You define a type alias:

```rust
type SharedSession = Arc<Session>
```

Now you want to write `impl` blocks on `SharedSession` and define a function. 

<!-- column: 1 -->

```rust
impl SharedSession {
    pub fn create_subscriber(&self) {}
}
```

This won't work, since `Arc` is foreign (it is in the standard library). 

You need to introduce a local trait first.

```rust
impl Subscribable for SharedSession {
    pub fn create_subscriber(&self) {}
}
```
---

### How do you use traits?

Designing traits is a very sensitive subject.

The way I like to structure my traits:

- **Minimal**: just the methods you need, no less than 2, no more than 5, but 
  - avoid single-method traits since traits add some boilerplate. 
  - if you discover "more minimal than others" methods, use super-traits or combined super-traits (see later) 
- **Composable**: methods that are  **not locally or temporaly dependent** on each others execution throughout the life-time of the program, should go into disjoint traits and used as combined super-traits.

How do you do your traits?

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

Algebraic data types
- Structs
- Enums

Generic standard library data types

- Collections: `HashMap`
- Result
- Option 
- Future
  
<!-- column: 1 -->

### Algebraic data types




---

### Polymorphic functions

Functions that take a type parameter.

- Can be free-standing (outside `impl` blocks)
- Methods inside an `impl` block

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

Introduce a type parameter `T` and assign it to the name of the associated type `Item`.

It goes inside the body of the `impl`, **not in the header**.

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

## Subclassing

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

If you come from any other object-oriented programming language. You might want to override methods in super-traits.

This is how such a program would look.

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

impl SuperTrait for char { }
impl SubTrait for char { }
```


<!-- column: 1 -->

Then you call 
```rust
fn main() {
    '?'.method()
}
```

It will not compile.


Explicit qualification of the trait is necessary.

**Question**: Why? 

<!-- pause -->

**Answer**: To keep behaviour of complex code with many super-traits predictable. Traits can only **extend supertraits** with methods that are not yet in the super-trait.

---

## How to organize super-traits?

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Question**: How do you use super-traits?

<!-- pause -->

### Use case 1: Repeated trait bounds

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

### Lifetimes in associated types

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

**Question**: Give a nice definition of a function.

<!-- pause -->

**Answer**: Things that **produce values**.

### Traditional function types

Rust has a few function-like things:
- Function items
- Function pointers
- Closures

### What else?

At the end of the presentation we will see other magical things that can produce values...

---

## Function traits

Function items, pointers and closures can be categorized **using the trait system** of Rust.


1. `Fn`: closures that do not capture mutable references and may be called infinitely
2. `FnMut`: closures that may reference mutable variables
3. `FnOnce`: closures that can only be called once and may move or drop variables

Function items and function pointers **satisfy all these traits**.

Closures form a ladder: `Fn => FnMut => FnOnce`. Closures will be discussed after function items.

---

## Function items

Named function declarations that implement all function traits `Fn, FnMut, FnOnce`. 

### Free functions

May be on top-level or nested as **helper functions**.

```rust
fn foo<T>() { }
```

Methods are preferred over free functions.

### Methods

Methods defined in `impl` blocks of traits on concrete types also become function items.


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

The keyword `self` can be of any type derived from the type `Self`.

Methods group related functions together and allow us to chain them. 

**Chaining methods** improves visibility compared to function composition of free functions.

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

If there are multiple input lifetime positions, but one of them is `&self` or `&mut self`, the lifetime of the reference to `self` is assigned implicitly to all elided output lifetimes.


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

**Question**: How can we assign function items with compatible signatures?

<!-- pause -->

**Answer**: Cast them as function pointers (see in a few slides).

Recursion in function items is possible but discouraged because no tail-call optimization happens and stack overflow may happen easily.

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

The implementation of closures is taken for granted in most dynamic languages. Many low-level languages don't have them. Rust strikes a balance, so it is important to know how closures work under the hood.

The **body block of the closure is analyzed**: 
1. variables in the body that refer to variables in the surrounding scope are marked as captured
2. `struct` generated at compile time with as fields the references to the captured variables, it serves as the environment for the closure body
 
The generated `struct` is **invisible**. Consequences:
- The direct type of a closure is implicit and cannot be used directly.
- Closures can only be referenced through their trait signatures (we will need `impl Trait`, see later)

This explains why closures are called **unnameable types** or **Voldemort types**.

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


For details about the construction, see [Finding closure in Rust](https://huonw.github.io/blog/2015/05/finding-closure-in-rust/)


Because closures are just `Environment` structs, they behave like structs.


---

## Bounds on closure structs

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

### Auto-trait bounds

Closures implement marker traits when captured content does: `Sync`, `Send`, `Copy`, `Clone`.

### Lifetime bounds

Closure may capture data that contains references, so they can have lifetime bounds.

Closures that are sent between async tasks have to be `'static`.

It is difficult to come up with non-artificial examples of non-trivial life-time bounds on closures.

**Question**: Who has an example?


---

## Variants of closures

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


We start at the bottom of the ladder with the least powerful closure.

### The base case `FnOnce`


The least a closure should be able to do is be called once = the **base case**.


Imagine the following closure 

```rust
let mut v = vec![];
let closure = move || v.push(1);
```

What does this compile to?

```rust
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


<!-- column: 1 -->

Notice how **capture-by-value** works:

1. we mark the closure as call by value with `move`.
2. the generated environment `struct` takes ownership of the local variable `v`. 

In the `struct` of a `FnOnce` closure the signature of `call` is written as `call(self)` which means that it's type is `Environment -> ()`. 

In general, the body of a `FnOnce` closure may:

- mutate captured mutable variables or references.
- move references or variables captured by value with `move` into new structs
- drop variables captured by value with `move`

What can closures **not do**: drop things that are referenced elsewhere.

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

---

### Closures that are `FnMut`

The signature of the `call` method on the implicit struct is `call(&mut self)`.

The struct may have
- mutable references to its containing scope (but importantly, does **not need to**)
- immutable references to its containing scope


```rust
let mut x = 5;
{
    let mut square_x = || x *= x;
    square_x();
}
assert_eq!(x, 25);
```

Compared to `FnOnce`, a `FnMut` closure:
- cannot consume or move out captured variables
- can be called more than once for sure, so it must implement `FnOnce`
- can only be called once at a time, 


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
- faster then trait objects of function traits,`dyn Fn` (see later for more information about trait objects)

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


## How should we handle complex types?

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Sometimes
- we don't know the type
  - the particular instance of a trait we exactly need as input or output for a function signature
  - the actual type is hidden from the user. These types are called unnameable or Voldemort types (for example closures)
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

## The  `impl` everywhere project

1. Signatures of methods in traits can use `impl Trait`, also known as "return position impl Trait" (RPIT) 
   - Implemented using a kind of associated types [See](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html)
2. Type aliases can use `impl Trait`: `type I = impl Iterator<Item = u32>`. [See](https://rust-lang.github.io/impl-trait-initiative/explainer/tait.html)
3. Traits with associated types can use `impl Trait` as bounds for the associated type (in nightly Rust)
4. In local variable bindings `let x: impl Future = foo()` (planned) 



[See](https://rust-lang.github.io/impl-trait-initiative/)


---

## Example: `impl` for associated types

To enable it, in crate root: `#![feature(impl_trait_in_assoc_type)]`

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

First we define a new trait 

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

(Notice we used the stabilised return position `impl Trait`)

<!-- column: 1 -->

Then we define something that may have a subset

```rust
struct SubsetWrapper<'a> {
    everything: &'a HashMap<usize, i32>,
    subset_ids: &'a [usize],
}
```

And we use the unstable syntax for `impl` bounds on associated types

```rust
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

## Impl on type aliases

```rust
mod odd {
    pub type OddIntegers = std::iter::Filter<std::ops::Range<u32>, /* what goes here? */>;

    pub fn odd_integers(start: u32, stop: u32) -> OddIntegers {
        (start..stop).filter(|i| i % 2 != 0)
    }
}
```

---

### Trait objects

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Every type system has **its limits**. 

There is only so much you can know **at compile-time**.

The solution to this problem is **trait objects**.

These are a way to handle things that satisfy some trait but we don't know anything else about them.


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

The trait and it's methods is all we have for a trait object. 


Notice the `dyn` keyword that declares the presence of a trait object. It is compulsory, since trait objects are handled completely different from `impl Trait`s.


<!-- column: 1 -->

### Unsizedness

Different things that implement the same trait, may have a different size. In theory, you could compute an upper bound on the size given all available implementing types, but that would be inefficient. Instead, allocation happens on the heap, dynamically, as needed.

In addition, Rust marks all trait objects as `!Sized`.

**Question**: What is the meaing of `!Sized` in this context "size may vary greatly at run-time".

<!-- pause -->

**Answer**: The`!Sized` means a trait object is not stored on the stack, but on the heap.

The standard way to deal with `T: !Sized` in function signatures is to 
- use a reference type such as `&T` or `&mut T` 
- or a heap-allocated indirection such as `Box` (then you get a reference through `as_ref()`, but I think this is done automatically).

So in practice when dealing with trait objects, you have to type your related functions with `&dyn Trait` to keep them as general as possible.

---

## Intermezzo: What else is `!Sized`?


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Trait objects are part of the family of **dynamically sized types** (DST) 

- Their type is unknown until run-time or varies greatly, so their size is assumed to be unknown until run-time.
- They cannot be allocated on the stack.

Every type parameter `D` in a generic type `GenericType<D>` receives the bound `D: Sized` by default. So, in general, we cannot pass trait object types as type parameters. 

If you want to include DSTs as possible type parameters, you have to use `?Sized` as in `GenericType<D: ?Sized>`.

<!-- column: 1 -->

### Examples of unsized types

**Question**: Give a few examples of unsized (`!Sized`) types that are not trait objects.

<!-- pause -->

**Answer**: Slices that are not behind a reference such as `str` or `[T]`.

### Fat pointers

All DSTs have something in common. They are called **fat pointers**.
  
**Question**: What are fat pointers? Why are they called fat pointers?

<!-- pause -->

**Answer**: A fat pointer contains a pointer plus some information that makes the DST "complete":
- the length
- a pointer to the method table
  

[See](https://stackoverflow.com/questions/57754901/what-is-a-fat-pointer)


---

## Dispatch of methods

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Additionally, there is some overhead when we call methods on trait objects. 

For all the `struct`s or types that implement the trait in the trait object, the compiler has to create a separate table (called a **vtable**) with a map 

```
method name -> function pointer
```

The table may be quite large and increase the size of the compiled binary. 

Each trait object stores a pointer to the right table at run-time.

Going from the value to the vtable and then to a function pointer **adds a certain amount of time**.

The lookup of methods through the pointer to the vtable and then inside the table is called **dynamic dispatch**.

The indirection **prevents optimization** by the compiler.

<!-- column: 1 -->

### Type erasure

**Important**: You can't magically extract information from a trait object that is not declared in the trait. 

The vtables only contain function pointers for methods declared in the trait or it's super-traits.

If you decide that the overhead of a trait object is justified you have to accept the fact you lose any concrete information such as fields of any implementing type. 

Code near trait objects should be able to deal with the methods of the trait and not depend on concrete types.

---

## Creating trait objects 

**Question**: Which of the following traits can be used to create a trait object?
- A trait that is `Sized`
- A trait that is `Clone`

<!-- pause -->

**Answer**: Neither of them. `Sized` is irrelevant since every trait object `dyn T` for `T: Sized` is `!Sized` anyway. Traits that are Clone need information about the size of `Self` to allocate on the stack. This information is not present at run-time. So they both cannot be used to create a trait object.

<!-- pause -->

### Object safe traits

There are rules that say which traits are **object safe** and can be turned into traits.

> You must be able to call methods of the trait the same way for any instance of the trait
> properties guarantee that, no matter what, 
> - the size and shape of the arguments and return value only depend on the bare trait — 
> - not on the instance (on Self) or any type arguments (which will have been "forgotten" by runtime)

[See](https://www.reddit.com/r/rust/comments/kw8p5v/what_does_it_mean_to_be_an_objectsafe_trait/)

---



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

## What else?

**Question**: These three function traits are nice, but which other things produce values?

<!-- pause -->

### Alternative functions

To be complete, there are also things with `call`-like functionality such as
- iterators
- co-routines and generators
- **async blocks** and futures
- **async iterators**
- never-ending or diverging functions (the **never** type `!`)
- fallible functions (`Result` monad)
- panicking functions
- unsafe blocks (code with possibly dangling pointers)

All these ways to produce values can be seen as **effects**: modes of computation or ways to have side-effects

--- 

## Effect syntax

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


Different effects introduce duplication. For example, there are similar traits: `AsyncRead` vs `Read`.

One proposal is

```rust
#[maybe(async)]
struct File { .. }

#[maybe(async)] 
impl File {
    #[maybe(async)]
    fn open<P>(p: P) -> Result<Self>
    where
        P: AsRef<#[maybe(async)] Path>;
}
```

See the blog post about Rust written by [Yoshua](https://blog.yoshuawuyts.com/extending-rusts-effect-system/)

<!-- column: 1 -->


The Rust syntax proposal is quite ugly compared to Koka

```koka
fun square1( x : int ) : total int   { x*x }
fun square2( x : int ) : console int { println( "a not so secret side-effect" ); x*x }
fun square3( x : int ) : div int     { x * square3( x ) }
fun square4( x : int ) : exn int     { throw( "oops" ); x*x }
```

See:

- [Koka](https://koka-lang.github.io/koka/doc/index.html)
- [Effekt](https://effekt-lang.org/)

Let's focus on Rust.

---

## Creating simple iterators

An iterator is the simplest kind of effect. It's side-effect is **advancing some internal state** incrementally.

Iterators are a useful intermediate data structure that are **easy to reason about** and handle.

In most languages iterators are written with the `yield` keyword. Rust goes for an approach with traits and introduces `next`.

### Rust

There are a few built-in constructors for iterators

- Bounded ranges `0..10`
- Unbounded ranges `..` (`RangeFull`)

Iterators can often be constructed from iterable data types with the `iter` or `into_iter` methods.

The most basic iterable data type is a **slice** (or also called an array), which is allocated on the stack. 

Slices give rise to different iterators

```rust
let slice = ['r', 'u', 's', 't'];
let mut iter = slice.windows(2);
assert_eq!(iter.next(), Some(&['r', 'u'][..]));
assert_eq!(iter.next(), Some(&['u', 's'][..]));
assert_eq!(iter.next(), Some(&['s', 't'][..]));
assert_eq!(iter.next(), None);
```


---

## Juggling with iterators

Iterators have many helper methods, called **adapters**..

```rust
let a = ["1", "two", "NaN", "four", "5"];

let mut iter = a.iter().filter_map(|s| s.parse().ok());

assert_eq!(iter.next(), Some(1));
assert_eq!(iter.next(), Some(5));
assert_eq!(iter.next(), None);
```

Clippy will usually detect when you need these adapters.

The method `size_hint` gives a lower and optional upper bound on the remaining items.

```rust
let a = [1, 2, 3];
let mut iter = a.iter();
assert_eq!((3, Some(3)), iter.size_hint());
let _ = iter.next();
assert_eq!((2, Some(2)), iter.size_hint());
```

**Question**: Which function uses the `size_hint`?

<!-- pause -->

**Answer**: The `collect` function needs to know how much empty space to allocate.


If you are not satisfied, look at [itertools](https://docs.rs/itertools/latest/itertools/).

---


### Ranges and FnOnce

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

#### Question

A difficult one.

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

1. The inner part `|| .. .method()` is parsed as `(|| ..).method()`
...
<!-- pause -->

2. The type of `|| ..` is `FnOnce() -> T` where `T` is inferred to be `RangeFull`
3. We resolve the of the `(|| ..).method()` to the method in `impl<F: FnOnce() -> T, T> Trait for F`.
4. It prints `2` and returns a closure.
5. `(|| .. .method())()` evaluates the closure and returns `4`

[See](https://dtolnay.github.io/rust-quiz/33)

---

## Implementing `IntoIter`


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

```rust
struct Grid {
    x_coords: Vec<u32>,
    y_coords: Vec<u32>,
}


struct GridIter {
    grid: Grid,
    i: usize,
    j: usize,
}

impl Iterator for GridIter {
    type Item = (u32, u32);

    fn next(&mut self) -> Option<(u32, u32)> {
        if self.i >= self.grid.x_coords.len() {
            self.i = 0;
            self.j += 1;
            if self.j >= self.grid.y_coords.len() {
                return None;
            }
        }
        let res = Some((self.grid.x_coords[self.i], self.grid.y_coords[self.j]));
        self.i += 1;
        res
    }
}
```

<!-- column: 1 -->

```rust
impl IntoIterator for Grid {
    type Item = (u32, u32);
    type IntoIter = GridIter;
    fn into_iter(self) -> GridIter {
        GridIter { grid: self, i: 0, j: 0 }
    }
}


fn main() {
    let grid = Grid { x_coords: vec![3, 5, 7, 9], y_coords: vec![10, 20, 30, 40] };
    for (x, y) in grid {
        println!("point = {x}, {y}");
    }
}
```

See [Google](https://google.github.io/comprehensive-rust/iterators/intoiterator.html)

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
- The iterator returned by `into_iter` on `T` or `&T` may yield any of `T`, `&T` or `&mut T`, depending on the context.
- The iterator returned by iter on `T` will always yield references.
<!-- pause -->

### For loops

**Question**: Does a for loop use `into_iter` or `iter`?

<!-- pause -->

**Answer**: `into_iter`

<!-- column: 1 -->

A `for x in it` where `it: IntoIterator` desugars to:

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

# Errors

---

## Bubbling errors up


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

A common decision developers have to make is whether to **bubble up** errors.

### The `Result` type

```rust
fn halves_if_even(i: i32) -> Result<i32, Error> {
    if i % 2 == 0 {
        Ok(i / 2)
    } else {
        Err(/* something */)
    }
}

fn do_the_thing(i: i32) -> Result<i32, Error> {
    let i = match halves_if_even(i) {
        Ok(i) => i,
        Err(e) => return Err(e),
    };

    // use `i`
}
```

The syntax in Rust to bubble up errors is with the `?` operator:

```rust
fn do_the_thing(i: i32) -> Result<i32, Error> {
    let i = halves_if_even(i)?;

    // use `i`
}
```
---

### Option 

Another example the `Option` type. 

The position function returns `None` if no match is found. 

```rust 
fn find_element(data: Vec<i32>, target: i32) -> Option<usize> {
    let index = data.iter().position(|&x| x == target)?;
    // do something with index
    Some(index)
}
```

The question mark propagates the inner `None` to an outer `None`.

**Question**: What do `Result` and `Option` have in common?

<!-- pause -->

**Answer**: In both cases we start inside with an intermediate value and we decide halfway based on the intermediate value if we should return early.  

---

## The fallibility effect 

The question mark operator introduces a new way to change the execution flow of programs.

It is a kind of **effect** since it adds something new to the way we compute.

In other languages, fallibility is not as safe in Rust and you have to explicitly **catch throws**.

In the following slides we will discover how fallibility is implemented in Rust.

---

## Starting from infallible functions

We  may start with the usual infallible function.

A **fold** is an operation that folds succeeding elements onto eachother. It never fails.

```rust
fn simple_fold<A, T>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> A,
) -> A {
    for x in iter {
        accum = f(accum, x);
    }
    accum
}
```

The idea is clear. We accumulate the output of the accumulation function `f`.

What if the accumulate function may fail and we want to **bubble up** the error?

The signature of `f` will have to change.


---

## Creating a fallible fold


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

The standard library of Rust introduces a new trait called `Try` that has two methods:

- `from_residual`: how to bubble up errors.
- `from_output`: how to bubble up success.

<!-- pause -->

We can call these methods on a type `R` that implements the `Try` trait. 



```rust
pub fn explicit_fallible_fold<A, T, R: Try<Output = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    for x in iter {
        let cf = f(accum, x).branch();
        match cf {
            ControlFlow::Continue(a) => accum = a,
            ControlFlow::Break(r) => return R::from_residual(r),
        }
    }
    R::from_output(accum)
}
```


This is a very general form and very verbose.


<!-- column: 1 -->

How can we simplify this? Remember we have the `?` operator.

<!-- pause -->


The `?` operator adds some useful syntax sugar and we can compress the match expression.

```rust
fn simple_try_fold<A, T, R: Try<Output = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    for x in iter {
        accum = f(accum, x)?;
    }
    R::from_output(accum)
}
```

Notice how the inner early `return` statement became invisible. 

This new strange syntax **adds complexity** for newcomers to Rust.

---

### What is `Try` actually?

<!-- column_layout: [1, 1] -->

<!-- column:   0 -->

Let's take a closer look at the trait `Try` that `?` uses.

We need an `unstable` feature `try_trait_v2`.

Formally, the `Try` trait looks as follows (simplified):

```rust
pub trait Try {
    type Output;
    type Residual;

    fn branch(self) -> ControlFlow<Self::Residual, Self::Output>;

    fn from_residual(residual: Self::Residual) -> Self;
    fn from_output(output: Self::Output) -> Self;
}
```

When `R: Try` we could say that `R` stands for an intermediate computation.

- The associated type `Residual` is the **inner break type**, the type associated with an inner failed computation that has to be bubbled up. `R` is the type of the outer failed computation, the result after bubbling up.
- The associated type `Output` is an **inner success type**, the success that has to be bubbled up. It can be converted to `R` with `from_output`.

<!-- column: 1 -->


The `branch` method is the core of the trait. 

1. We are in the middle of the computation and encounter a fallible function
2. We execute the fallible function and we obtain an `R` (can be thought of as a `Result`) 
3. We see the operator `?` which is syntax sugar for conditional return

At this point we arrive at the core of the `Try` trait.

1. `?` will trigger the evaluation of `branch` on `R`
2. The custom implementation of `branch` will return break or continue.

See [std](https://doc.rust-lang.org/std/ops/trait.Try.html)

---

## Going back to our fold

<!-- column_layout: [1, 1] -->

<!-- column:   0 -->

```rust
pub fn explicit_fallible_fold<A, T, R: Try<Output = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    for x in iter {
        let cf = f(accum, x).branch();
        match cf {
            ControlFlow::Continue(a) => accum = a,
            ControlFlow::Break(r) => return R::from_residual(r),
        }
    }
    R::from_output(accum)
}
```

<!-- column: 1 -->

There are two options:
- **continue** with some value `a` of type `Output`.  
  - `a` becomes the output of the the `?` operator 
- **break** with some value of type `Residual`. 
  - The residual is converted to some `R` (an `Option` or a `Result`)
  - We never reach the location behind `?`
  - The converted residual `R` is returned

---

## `Result` as `Try`

<!-- column_layout: [1, 1] -->

<!-- column:   0 -->


The most common `Try` type.

The `Result<Success,Error>` type has as associated `Try` types:
- `Output` = `Success`
- `Residual` = `Self`

### Branching

The `Ok` variant of `Result` will trigger a `Continue` with the inner output value.

```rust
assert_eq!(Ok::<_, String>(3).branch(), ControlFlow::Continue(3));
```

A branch of an `Err` variant of the `Result` type return a `Break` that contains an instance of the associated `Residual` type.

```rust
assert_eq!(Err::<String, _>(3).branch(), ControlFlow::Break(Err(3)));
```

<!-- column:   1 -->

### Early return

The content of the `Break` is passed to the `from_residual` function

```rust
assert_eq!(Result::<String, i64>::from_residual(Err(3_u8)), Err(3));
```

### Final success

At the end `Success` is converted into `Result<Success, Error>` through the `from_output` function.

```rust
assert_eq!(<Result<_, String> as Try>::from_output(3), Ok(3));
```

---

## `Option` as `Try`

<!-- column_layout: [1, 1] -->

<!-- column:   0 -->

The associated types of `Option<T>>` are:
- `Output` = `T`
- `Residual` = `Self`

### Branching

```rust
assert_eq!(Some(3).branch(), ControlFlow::Continue(3));
assert_eq!(None::<String>.branch(), ControlFlow::Break(None));
```

### Early return

If we have a `Break`, the inner `None` is passed to `from_residual`:

```rust
assert_eq!(Option::<String>::from_residual(None), None);
```

<!-- column: 1 -->

### Successful finish

At the end `T` is converted into `R = Option<T>` through the `from_output` function: 

```rust
assert_eq!(<Option<_> as Try>::from_output(4), Some(4));
```

### Short-circuiting

Evaluating the `?` operator will trigger the branch function and then output a value of associated type `Output = T`:

```rust
assert_eq!(Option::from_output(4)?, 4);
```

--- 

## What's next?

Coroutines, coroutine implementations, async blocks, futures, async run-times.


---

### Questions?


You can always email me at willemvanhulle@gmail.com.

See also the great book "Programming Rust" by Blandy (2021).


This presentation was made using Markdown and a Rust tool [`presenterm`](https://github.com/mfontanini/presenterm).

