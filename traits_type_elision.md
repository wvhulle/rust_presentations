---
title: Advanced Rust
sub_title: Traits, closures and type elision
author: Willem Vanhulle
options:
  end_slide_shorthand: true
---

![otiv](otiv.jpg)

--- 
### Function pointers in methods 

<!-- column_layout: [2, 1] -->

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

#### Answer

```
1
```


<!-- pause -->


#### Long answer

A call that looks like `.f()` always resolves to a method, in this case the inherent method `S::f`. 


To call the function pointer stored in field `f`, we would need to write parentheses around the field access: 

```rust
fn main() {
    let print2 = || print!("2");
    (S { f: print2 }.f)();
}
```

---

## Traits

Semantics:
- A **trait** is a set of common behaviours of concrete types

Syntax:
- A list of methods
- use or reference one type of `self`

```rust
trait Animal {
    // Associated function signature; `Self` refers to the implementor type.
    fn new(name: &'static str) -> Self;

    // Method signatures; these will return a string.
    fn name(&self) -> &'static str;
    fn noise(&self) -> &'static str;

    // Traits can provide default method definitions.
    fn talk(&self) {
        println!("{} says {}", self.name(), self.noise());
    }
}
```


Synonyms:
- Haskell: typeclass
- TypeScript: interfaces


---

### Usage of traits

Traits should be:
- Atomic: minimal amount of methods
- Composable: different traits complement eachother


> "Traits: Composable Units of Behaviour", Sharli (2003)
---

### Inherent methods and priority

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

#### Question

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

impl Trait for S {
    fn f(&self) {
        print!("2");
    }

    fn g(&self) {
        print!("2");
    }
}
```

<!-- column: 1 -->


```rust
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


`S.f()` calls the inherent method `f`. If an inherent method and a trait method have the same name and receiver type, plain method call syntax will always prefer the inherent method. The caller would need to write `Trait::f(&S)` or `<S as Trait>::f(&S)` in order to call the trait method.

On the other hand, S.g() calls the trait method g. Auto-ref during method resolution always prefers making something into & over making it into &mut where either one would work.

---

### Auto-referencing

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->



#### Question

```rust 
trait Trait: Sized {
    fn is_reference(self) -> bool;
}

impl<'a, T> Trait for &'a T {
    fn is_reference(self) -> bool {
        true
    }
}

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

<!-- pause -->


#### Short answer

```
10
```

<!-- pause -->


#### Long answer

1. no implementation of Trait for an integer type that we could call directly. Method resolution inserts an auto-ref, effectively evaluating `(&0).is_reference`
2. Trait impls anywhere in a program are always in scope


[See](https://dtolnay.github.io/rust-quiz/14)

---

### Method resolution


<!-- column_layout: [2, 1] -->

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

`x` must be of type `&Self` for some type `Self` that implements `Trait`. We find that inferring `0: u32` satisfies both the constraint that `u32` is an integer as well as `u32` implements `Trait`, so the method call ends up calling `<u32 as Trait>::f(x)`.

[See](https://dtolnay.github.io/rust-quiz/15)

---


## Generic data types

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->


### Slices


### Structs



### Enums




<!-- column: 1 -->



---


## Polymorphism

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->


```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {result}");

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {result}");
}
```

<!-- column: 1 -->


Advantages of generic data types:

- Less duplication of code
- Easier to maintain



Disadvantages of generic structs:

<!-- pause -->

- You have to specify all the type parameters at the beginning of `impl` blocks.
- You have to find out how to supply type parameters to method definitions


---

### Methods for generic data types

Syntax: `impl<A>` blocks.
- All type parameters of the struct being implement should be declared
- Declaring more type parameters than present gives an error


---


## Traits with type parameters

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Semantics:
- Group methods that are not about a particular struct but about a whole class of structs


Syntax:
- A list of type parameters (generics) is declared 
- Type parameters can take defaults, this is useful for traits related to binary operator:

Example:

```rust
pub trait PartialEq<Rhs = Self> {
    fn eq(&self, other: &Rhs) -> bool;
    fn ne(&self, other: &Rhs) -> bool { !self.eq(other) }
}

struct Point {
    x: i32,
    y: i32,
}

impl PartialEq<(i32, i32)> for Point {
    fn eq(&self, other: &(i32, i32)) -> bool {
        self.x == other.0 && self.y == other.1
    }
}

let p = Point { x: 1, y: 2 };
let result = p == (1, 2);  // true

```
<!-- column: 1 -->


Advantages:
- Deduplicate code


---


### Traits with associated types

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

A 1-to-1 map from a generic type to its generic type parameters.

Move the declaration of the type parameter below the name of the trait: 

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```
<!-- pause -->


```rust
struct Buffer<T> { data: Vec<T> }

impl<T> Buffer<T> {
    fn new(data: Vec<T>) -> Self {
        Buffer { data }
    }
}

impl<T> IntoIterator for Buffer<T> {
    type Item = T; 
    type IntoIter = std::vec::IntoIter<T>; 

    fn into_iter(self) -> Self::IntoIter {
        self.data.into_iter()
    }
}
```


<!-- column: 1  -->

<!-- pause -->

One more example:

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

<!-- pause -->

Benefits
- When you reference such a trait as a type bound, you are not required to specify all the associated types, only the ones that you need
- You can use the names of the associated types instead of their order of definition when you specify them

Disadvantages
- Not all generic traits can be rewritten as traits with associated types.
- You have to use the names of the associated types
---


## Generic traits with associated types


Some traits have both:
- generic type parameters 
- associated types

```rust
pub trait Add<Rhs = Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```

---

## Trait bounds

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->


### Position of trait bounds

Every generic type parameter can be constrained by **trait bounds**.


- inline: `GenericType<SomeGenericParameter: SomeTrait>`
- When constraints become to complicated you should move them to `where` blocks: `where SomeGenericParameter: SomeTrait`


```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point<T: Add<Output = T>> {
    x: T,
    y: T,
}

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

<!-- column: 1 -->

```txt
error[E0277]: cannot add `T` to `T`
 --> src/main.rs:9:17
  |
9 | impl<T> Add for Point<T> {
  |                 ^^^^^^^^ no implementation for `T + T`
  |
note: required by a bound in `Point`
 --> src/main.rs:4:17
  |
4 | struct Point<T: Add<Output = T>> {
  |                 ^^^^^^^^^^^^^^^ required by this bound in `Point`
help: consider restricting type parameter `T`
  |
9 | impl<T: std::ops::Add> Add for Point<T> {
  |       +++++++++++++++
```


Be careful:
- If you add trait bounds to generic structs, you have to specificy them in each `impl` block.
- Same for traits



---

### Bad solution

```rust
impl<T: std::ops::Add> Add for Point<T> {
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

Still not good

```rust
impl<T: std::ops::Add<Output = T>> Add for Point<T> {
    type Output = Self;

    fn add(self, other: Self) -> Self::Output {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
```

Now we have both a long constraint on the struct and all the `impl` blocks.
- Constraints apply to all the method definitions
- We cannot easily add other functions to the `impl` blocks since they have to satisfy all the added constraints

---

## Blanket implementations

Define a new generic struct

```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point<T> {
    x: T,
    y: T,
}
```

Instead of adding the constraints to `Point`, we just add them to an `impl`.

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

assert_eq!(Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
           Point { x: 3, y: 3 });
```

This is called a **blanket implementation** .

Visible throughout all the crates, but can not be applied to purely foreign types

If you start repeating the bounds in different blanket implementations, it might be time to put all the repeated bounds in  a new trait  

---


### Fixing violated type bounds

If some type bound is not satisfied, the compiler will give an error.

How to fix the error:


<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Are you in an `impl` block outside of a trait? 

- Look at the contraints of the `impl` block.
   - They are in front of the impl block or behind in a where clause.
   - They are unordered, so do not verify the order
- Verify the contraints of the struct.
  - Compare the order of
     - the generic type parameters in the `struct` declaration
     - the type generics in the `impl`: `impl<A,B> for Struct<A,B>`. 
   - Look in the where clauses around the struct for unsatisfied bounds. They are unordered.
- Always break down combined trait bounds and check them individually.

<!-- column: 1 -->

Are you in an impl block in a `trait`?

- Do the same as outside an `impl` block.
- Look at the constraints in the trait definition.
- Look at the constraints imposed by supertrait bounds.

---

## Supertraits

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

```rust
trait Person {
    fn name(&self) -> String;
}

// Person is a supertrait of Student.
// Implementing Student requires you to also impl Person.
trait Student: Person {
    fn university(&self) -> String;
}

trait Programmer {
    fn fav_language(&self) -> String;
}
```

<!-- pause -->

<!-- column: 1 -->


`CompSciStudent` (computer science student) is a subtrait of both Programmer and Student. 


Implementing `CompSciStudent` requires you to impl both supertraits.

```rust
trait CompSciStudent: Programmer + Student {
    fn git_username(&self) -> String;
}
```

### Inheritance

Super traits allow to inherit methods from another trait: `Programmer + Student`

Since you can inherit from multiple super traits it is a kind of **multiple inheritance**

---

## Super traits and method resolution

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

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

Overriding methods won't work. This rule is to keep code predictable. Traits can only **extend supertraits**.

---

## Moving bounds to super traits

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Sometimes trait bounds are repeated in different `impl` blocks.

Solution:

1. Create a new trait with a useful name
2. Put the repeated bounds as super traits

If you do this, you don't need to specify the bounds ever again besides specifying the subtrait.

### Associated types in super traits


Super traits that have associated types can also have bounds on the associated types.

For example

```rust
trait SubTrait: AssociatedSuperTrait<State: Sync> {}
```



<!-- pause -->

<!-- column: 1 -->

You can even reference a new associated type in the `SubTrait`:

```rust
trait SubTrait: AssociatedSuperTrait<State=Self::SubState> {}
```

If the associated type with name `State` has a lifetime parameter you can specify it with

```rust
trait SubTrait: for<'a> AssociatedSuperTrait<State<'a>: Sync> {}
``` 
---

### Multi-threading bounds

Auto-traits are traits that are implemented automatically by the compiler. 

The most common trait bounds.

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

#### Send

Definition: 
- “safe to be move between threads”
- **thread safe**

Semantics:
  - transfer ownership to other thread
  - other thread becomes responsible for dropping object
  - share a mutable reference to another thread

Automatically implemented by compiler based on some rules (**auto-trait**)

<!-- column: 1 -->

Examples:
- Structs without references are `Send + 'static`
- Structs with fields that are references with lower-bound lifetime parameter `'a` are `Send + 'a`
- `Cell`

Counterexamples:
- `*mut T`

---

#### Sync

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Definition: `&T` is Send.

Semantics: safe to access immutably from several threads in parallel

Rules
- Pointers are in one-to-one corresponds with pointers to pointers (`&T = &TT`). This implies that `T : Sync <=> &'_ T : Sync`.
- A consequence of this is that if `T: !Sync` then `&'_ T: !Sync`

<!-- column: 1 -->


Examples: `Arc`

Counterexamples:
- `Rc`
- `RefCell`.

---

### Inheritance of `Send` and `Sync`

**Question**: in which cases does `Arc<T>` implement Send?

<!-- pause -->

**Answer**: `T` must implement `Send`.

Why?

<!-- pause -->

`Arc<T>` may move `T` to another thread if the last handle is on another thread.

<!-- pause -->


**Question**: in which cases does `Arc<T>` implement Sync?
<!-- pause -->

Answer: `T` must implement `Sync`.


---

## Combinations of `Sync` and `Send`

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

### `Send + !Sync`: 

Types that may be accessed from any thread but only one at a time
- `Cell`
  - set inner value
  - swap inner value
  - copy inner value
- `RefCell`
  - removes compile-time borrow checks
  - may panic

### `!Send + !Sync` 

Types that manipulate shared handles to `!Sync` types

- `Rc<T>`
- Raw pointers


<!-- column: 1 -->

### `!Send + Sync`:

Rare combination: may be accessed immutably from any thread and multiple in parallel

Example: `MutexGuard` of a `T`
- cannot be dropped in a different thread to prevent data races so it is not `Send`
- If `T` is sync then it follows from `T : Sync ⇔ &'_ T : Sync` that a `MutexGuard<T>` is Sync


[See](https://users.rust-lang.org/t/example-of-a-type-that-is-not-send/59835/3)

<!-- reset_layout -->


[See](https://stackoverflow.com/questions/68704717/is-the-sync-trait-a-strict-subset-of-the-send-trait-what-implements-sync-withou)

---

## Lifetime bounds

Structs or types can contain references. In that case they receive a lifetime parameter. 


<!-- pause -->

**Question**: What does `T: 'a` mean?

<!-- pause -->

**Answer**: You can hold on `T` 
- for the lifetime `'a` 
- until you move it out


<!-- pause -->

Rules/corollary:

- `&'a T => T: 'a`, since a reference to `T` of lifetime `'a` cannot be valid for `'a` if `T` itself is not valid for `'a`.




---

### Questions


**Question**: What does `T: 'static` mean?:

<!-- pause -->

**Answer**:
- `T` does not contain any lifetimes
- `T` is an owned variable

Every type parameter for values that have to be send between (worker) threads has to have the bound `'static`.

<!-- pause -->

**Question**: What is the difference between `T : 'static` and `&'static T`?


<!-- pause -->

**Answer**: 
- `T : 'static` means that `T` must not contain any non-`'static` references.
- `&'static T` means that this is a reference that lives forever.

---
## Function traits

Semantics:
- Fn: functions that can be called infinitely and may reference variables
- FnMut: functions that may reference mutable variables
- FnOnce: functions that can only be called once and may move or drop variables

Rules:
- `Fn => FnMut => FnOnce`
---
## Function items

- syntax
  - created with `fn foo() {}`
  - `fn(u32) -> i32 {fn_name}` in error messages
- semantics
  - stateless, pure
   - can be transferred to other threads since there is no risk of data races
  - references to immutable code in compiled binary
    - can easily be cloned or copied  
    - can be optimized away `=>` no pointer needed `=>` zero sized
  - have unique unnameable types
  - implement all the function traits

Every function (or every distinct instantiation of a generic function) has its own unique type. Even if they have the same signature.
- `fn(u32) -> i32 {fn_name}` in error messages

Type itself encodes the information of what function will be called. At runtime no function pointers are needed.



---
## Function pointers

Syntax
- declared with `fn() -> ()`
- not a trait
  - so not written with a capital letter.
  - cannot be used as a trait bound

Semantics
- Points to 
  - a top-level defined function item
  - an associated function
- cannot capture from the environment
- Implements all the following traits Fn, FnMut, and FnOnce
- size of a pointer, has to be dereferenced,
  - might be slower than calling a function item or closure directly,
  - faster then dyn Fn
- Primitive type, not a trait


---

### How to create function pointers


- How can it be created?
    - non-capturing closures
    - function items with the same signature
- How can it be used?
  - Can be passed as argument to
    - function items
    - other function pointers
  - Can be returned from a function item


When is it particularly useful?
- When using FFI with languages that don't support closures

---


<!-- column_layout: [2, 1] -->

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

<!-- column: 1 -->

#### Answer

```
0
```



Why?

<!-- pause -->

Function items are treated as compile-time constants.

---


<!-- column_layout: [2, 1] -->

<!-- column: 0 -->


### Question


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

<!-- column: 1 -->


Why?

<!-- pause -->


Function pointers are stored in a pointer-like form that can be used at run-time.

---

<!-- column_layout: [2, 1] -->

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



<!-- column_layout: [2, 1] -->

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

## Closures

A kind of function defined in a certain scope that may capture references to variables in the containing scope

Non-capturing closures can be converted into function pointers automatically

Closures cannot be `async` but they can return a `Pin<Box<dyn Future>>`.

Closures are also called
- Lambda-functions
- Anonymous functions

---



<!-- column_layout: [2, 1] -->

<!-- column: 0 -->


#### Question

```rust
struct S(i32);

impl std::ops::BitAnd<S> for () {
    type Output = ();

    fn bitand(self, rhs: S) {
        print!("{}", rhs.0);
    }
}

fn main() {
    let f = || ( () & S(1) );
    let g = || { () & S(2) };
    let h = || ( {} & S(3) );
    let i = || { {} & S(4) };
    f(); g(); h(); i();
}
```

<!-- column: 1 -->

<!-- pause -->

#### Short Answer

The output will be 123


<!-- pause -->

#### Long answer

```rust
let i = || {
    {}
    &S(4)
};
```
The combination of an inner and outer `{}`, makes the parser interpret the `{}` as an empty block statement followed by & as a reference. 


---
## Closures and lifetimes


<!-- column_layout: [2, 1] -->

<!-- column: 0 -->


Will the following code compile?

```rust
fn main() {
    let closure = |x: &i32| x;
}
```

<!-- pause -->

**Answer**: No, because lifetime elision rules are different for closures.


```
  |
2 |     let closure = |x: &i32| x;
  |                       -   - ^ returning this value requires that `'1` must outlive `'2`
  |                       |   |
  |                       |   return type of closure is &'2 i32
  |                       let's call the lifetime of this reference `'1`
```

<!-- column: 1 -->

The closure definition can be expanded to:

```rust 
fn main() {
    // input and output each get their own distinct lifetimes
    let closure = for<'a, 'b> |x: &'a i32| -> &'b i32 { x };
    // note: the above line is not valid syntax, but we need it for illustrative purposes
}
```

Two distinct lifetimes are created for the input and output type. For normal functions, only one lifetime is created.

---

## Closure implementations

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

How are they implemented?

- the body block of the closure is analyzed
- variables in the body that refer to variables in the surrounding scope are marked as captured
- struct generated at compile time with as fields the references to the captured variables, it serves as the environment for the closure body
  - the struct is invisible and out of reach for the programmer in normal Rust code
  - This makes closures part of the family of unnameable types (also called **Voldemort types**)

Only implement `Copy`, `Clone`,  `Send` and `Sync` when their contents (references or values) do.


Disadvantages
- The exact type of a closure struct cannot be written out to type an input or output argument 

<!-- column: 1 -->


```rust
let mut v = vec![];

// nice form
let closure = || v.push(1);

// explicit form
struct Environment<'v> {
    v: &'v mut Vec<i32>
}

// let's try implementing `Fn`
impl<'v> Fn() for Environment<'v> {
    fn call(&self) {
        self.v.push(1) // error: cannot borrow data mutably
    }
}
let closure = Environment { v: &mut v }
```

[See](https://huonw.github.io/blog/2015/05/finding-closure-in-rust/)


---
## Variants of closures

There are different kinds of closures based on the signature of their call method on the underlying struct:
- Fn:
    - the signature is `call(&self)`,
    - the body of the closure only may have
        - immutable references to its containing scope
        - values that were moved from the containing scope (and only use them by reference afterwards)
    - can be called from anywhere, multiple times
    - Must implement FnMut

---

### `FnMut`

All closures that may modify captured variables.

- the signature is `call(&mut self)`
- can have
    - mutable references to its containing scope
    - immutable references to its containing scope
- cannot consume or move out captured variables
- can be called more than once, but only once at a time, must implement `FnOnce`

```rust
let mut x = 5;
{
    let mut square_x = || x *= x;
    square_x();
}
assert_eq!(x, 25);
```

---

### `FnOnce`


A closure that can be called at most once

- signature is `call(self)`
- It may mutate captured variables or references
- May move variables that are moved in out
  - consume captured variables
  - apply functions to them and call by value, not by reference
  
This means that it is not `Fn` or `FnMut`, because those should be able to be called multiple times

Implemented by every closure (a closure that cannot be called once is not a closure)


---

### Move

You can change the default behaviour to only create references to the surrounding environment. You can also move variables inside the struct representing the environment of the closure. This is done by using the `move` keyword.
- takes ownership of variables in the surrounding scope
- inside the closure the captured and moved variables are used by reference

**Question**: is every `move` closure `FnOnce`.

<!-- pause -->

**Answer**: No, the captured references are moved inside, but the closure can be called multiple times since the body uses the captured variables by reference.

---

<!-- column_layout: [2, 1] -->

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

---
### Ranges and FnOnce



<!-- column_layout: [2, 1] -->

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



<!-- column_layout: [2, 1] -->

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



<!-- column_layout: [2, 1] -->

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

<!-- column_layout: [2, 1] -->

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

<!-- column_layout: [2, 1] -->

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

<!-- column_layout: [2, 1] -->

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


---

### Dynamic versus static dispatch

<!-- column_layout: [2, 1] -->

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

<!-- column_layout: [2, 1] -->

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

<!-- column_layout: [2, 1] -->

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

<!-- column_layout: [2, 1] -->

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

### `Impl Trait`

The other type of opaque types or type elision.

Syntax sugar for hardcoding a specific type that can be inferred by the compiler

Benefits:
- No extra heap allocation
- No dynamic dispatch overhead

Synonyms
- anonymous types

---

### Basic example

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

### `impl` in arguments

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

[See](https://doc.rust-lang.org/rust-by-example/trait/impl_trait.html)

---

### `impl` in return type


<!-- column_layout: [2, 1] -->

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


<!-- column_layout: [2, 1] -->

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


<!-- column_layout: [2, 1] -->

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