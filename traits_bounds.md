---
title: Advanced Rust
sub_title: Traits
author: Willem Vanhulle
options:
  end_slide_shorthand: true

---

## Contents

Mix of information and quizes (from Tolnay) about:

- Traits
- Generics
- Super-traits
- Common trait bounds
- Life-time bounds

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
- The type of `self` is `Self` by default, but it can also be any `Deref<Self>` such as `Rc<Self>`.
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

For some reason, it cannot be `Clone`.

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

## Summary of the `.` operator

Performs:
- auto-dereferencing `Deref<T>` types.
- auto-referencing (prefering immutable references)

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

Inspired from mathematical computer science.

Not every language has them:

- Product types = structs
- Sums = enums

Allow consistent pattern matching. 

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

The more type parameters that your structs or methods get and the deeper your architecture becomes, the more type parameters you have to specify in `impl` blocks. 

So structs and methods should take as few type parameters as possible.

<!-- pause -->

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
```

We `impl` both traits for `char` and use the default implementations.
```rust
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

<!-- pause -->

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

trait SubTrait: SuperTrait<State: Sync> { }
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

---

<!-- column: 1 -->

### The special life-time `'static`

If `T` does not contain any non-`'static` references, then `T` receives a default lifetime bound `T: 'static`. `'static` is a reserved name for a lifetime. 

<!-- pause -->

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

