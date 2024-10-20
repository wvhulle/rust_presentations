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


## Generic structs

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Synonyms
- Generics in Python
- Templates in C++



Semantics
- a struct that for which the fields have different types based on the type parameter

Syntax
- a placeholder type parameter that has to be specified before methods can be called

Implementation

<!-- pause -->


- The compiler applies monomorphization to code contain generics.
  

How to improve efficiency?

<!-- pause -->

- Jon Gjengset: "define inner non-generic functions"


<!-- column: 1 -->

Advantages of generic structs:

- Less duplication of code
- Easier to maintain


Disadvantages of generic structs:

<!-- pause -->

- You have to specify all the type parameters at the beginning of `impl` blocks.
- All the type parameters need to be constrained by traits
- You have to find out how to supply type parameters to method definitions

---


## Generic method definitions

Syntax: `impl<A>` blocks.
- All type parameters of the struct being implement should be declared
- Type parameters can be constrained
- Constraints can be moved to a `where` block
- Constraints apply to all the method definitions
- If a generic parameter only applies to a function, you should move it to the mothod. Otherwise you will get `unconstraint type parameter` compiler errors.

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

<!-- pause -->

## Trait bounds

Type parameters in trait definitions may have constraints. At compile time:
- Perform type inference = checking whether the program is well-typed
- Try to compute concrete constructable types that satisfy the trait bounds. 

### Where to write bounds

Trait bounds can appear in


- inline: `GenericType<SomeGenericParameter: SomeTrait>`
- where clauses: `where SomeGenericParameter: SomeTrait`

Disadvantages:
- Each time you implement the trait inside `impl` blocks you have to include all the trait bounds from the trait definition


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



## Conditional implementations

Define a new generic struct

```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point<T> {
    x: T,
    y: T,
}
```

Restrict the `T` this implementation applies to.

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

Notice the **named** associated type

---

### Supertraits

Inherit methods or properties from another trait

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

<!-- column: 1 -->


CompSciStudent (computer science student) is a subtrait of both Programmer and Student. 

Implementing CompSciStudent requires you to impl both supertraits.

```rust
trait CompSciStudent: Programmer + Student {
    fn git_username(&self) -> String;
}
```

A kind of **multiple inheritance**

---

### Method resolution

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

Rust does not allow ambiguous method names


```
   |
24 |     '?'.method()
   |         ^^^^^^ multiple `method` found
```

Explicit qualification of the trait is necessary.

---

### Inheritance in Rust

Coming from object-oriented languages, you might want to repeat patterns.

Design patterns such a multiple inheritance work different in Rust.

Simply replacing generic structs by generic traits is not enough.

1. Analyze structs
2. Look for shared functionality
3. Decompose functionality in orthogonal traits
4. Use multiple supertraits for multiple inheritance

---

### Fixing violated type bounds

If some type bound is not satisfied, the compiler will give an error.

How to fix the error:


<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Are you in an impl block outside of a trait? 

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

Are you in an impl block in a trait?

- Do the same as outside an `impl` block.
- Look at the constraints in the trait definition.
- Look at the constraints imposed by supertrait bounds.

---

### Multi-threading bounds

Auto-traits are traits that are implemented automatically by the compiler. They are the most common trait bounds. Examples:

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

- automatically implemented by compiler based on some rules

Examples:
- Structs without references are `Send + 'static`
- Structs with fields that are references with lower-bound lifetime parameter `'a` are `Send + 'a`
- `Cell`

Counterexamples:
- `*mut T`


<!-- column: 1 -->
#### Sync

Definition: `&T` is Send.

Semantics: safe to access immutably from several threads in parallel

Rules
- Pointers are in one-to-one corresponds with pointers to pointers (`&T = &TT`). This implies that `T : Sync <=> &'_ T : Sync`.
- A consequence of this is that if `T: !Sync` then `&'_ T: !Sync`

Examples: `Arc`

Counterexamples:
- `Rc`
- `RefCell`.

---

### Inheritance of Sync

Question: in which cases does `Arc<T>` implement Send?

<!-- pause -->

Answer: `T` must implement `Send`.

Why?

<!-- pause -->

`Arc<T>` may move `T` to another thread if the last handle is on another thread.

<!-- pause -->


Question: in which cases does `Arc<T>` implement Sync?
<!-- pause -->

Answer: `T` must implement `Sync`.


---

### Combinations

<!-- column_layout: [2, 1] -->

<!-- column: 0 -->

Common combinations
- `Send + !Sync`: 
  - Semantics: may be accessed from any thread but only one at a time
  - Examples
    - `Cell`
      - set inner value
      - swap inner value
      - copy inner value
    - `RefCell`
      - removes compile-time borrow checks
      - may panic
- `!Send + !Sync`, 
  - Semantics: types that manipulate shared handles to `!Sync` types
  - Examples
    - `Rc<T>`
    - Raw pointers


<!-- column: 1 -->


Rare combination: `!Send + Sync`:

Semantics
- may be accessed immutably from any thread and multiple in parallel
- mutable access has to happen on the thread it was created on
- transferring a &mut T to another thread is not possible since it would break the protections that !Send has

Example: `MutexGuard` of a `T` [See](https://users.rust-lang.org/t/example-of-a-type-that-is-not-send/59835/3)
- cannot be dropped in a different thread, so it is not Send
- If `T` is sync then it follows from `T : Sync ⇔ &'_ T : Sync` that a `MutexGuard<T>` is Sync

<!-- reset_layout -->


[See](https://stackoverflow.com/questions/68704717/is-the-sync-trait-a-strict-subset-of-the-send-trait-what-implements-sync-withou)

---


## Lifetime bounds

Structs or types can contain references. In that case they receive a lifetime parameter. 

Syntax:
 
- Structs: lifetime parameters are written before the generic type arguments. 
- Functions: lifetime parameters are written before the argument list 

<!-- pause -->

**Question**: What does `T: 'a` mean?

<!-- pause -->

**Answer**: You can hold on `T` 
- for the lifetime `'a` 
- until you move it out


<!-- pause -->

Rules/corollary:

- `&'a T => T: 'a`, since a reference to T of lifetime 'a cannot be valid for 'a if T itself is not valid for 'a


<!-- pause -->

**Question**: What does `T: 'static` mean?:

<!-- pause -->

**Answer**:
- `T` does not contain any lifetimes
- `T` is an owned variable

Every type parameter for values that have to be send between (worker) threads has to have the bound `'static`.

---

### Questions

**Question**: What is the difference between `T : 'static` and `&'static T`?


<!-- pause -->

**Answer**: 
- `T : 'static` means that `T` must not contain any non-`'static` references.
- `&'static T` means that this is a reference that lives forever.



<!-- pause -->

**Question**: What is the difference between `let s = &'static "bla"` and `let s = "bla".to_string()`

<!-- pause -->

**Answer**:
- `let s = &'static "bla"` is hardcoded in the binary, cannot be declared mutable,  cannot be moved
- `let s = "bla".to_string()` is allocated on the heap, can be declared mutable
---
## Lifetime bounds in methods

If you call a method on a struct that accesses a field and put it in a reference, the struct is borrowed for the duration of the reference to the field. 

Solution: create copies instead of borrowing a reference.

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

### Size of function items

#### Question

Predict the output of

```rust
fn my_function(x: i32) -> i32 {
    x + 1
}

fn main() {
    let f = my_function; // f is a function item (ZST)
    println!("Size of f: {}", std::mem::size_of_val(&f));
}
```

#### Answer

```
0
```

---
## Function pointers

- Syntax
    - declared with `fn() -> ()`
    - not a trait
      - so not written with a capital letter.
      - cannot be used as a trait bound
- Semantics
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

### Size of function pointers


```rust
fn my_function(x: i32) -> i32 {
    x + 1
}

fn main() {
    let f: fn(i32) -> i32 = my_function; // f is a function pointer
    println!("Size of f: {}", std::mem::size_of_val(&f)); // Output: 8 (on 64-bit)
}
```

---

https://dtolnay.github.io/rust-quiz/34

#### Question

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

#### Short answer

```
20
```

#### Long answer

The first call in main coerces `a::<u8>` from a function to a function pointer (`fn(fn(u8)) {a::<u8>}` to `fn(fn(u8))`) prior to calling d, so its size would be 8 on a system with 64-bit function pointers
The second call in main does not involve function pointers; d is directly called with T being the inexpressible type of `a::<u8>`, which is zero-sized.

---

### Function pointer fields


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

#### Short answer

```
1
```


#### Long answer

A call that looks like .f() always resolves to a method, in this case the inherent method S::f. To call the function pointer stored in field f, we would need to write parentheses around the field access: 

```rust
fn main() {
    let print2 = || print!("2");
    (S { f: print2 }.f)();
}
```
---
## Closures


- Alternative names
  - Lambda
- Semantics
  - A function defined in a certain scope that captures reference to variables in the containing scope
  - Creates an implicit struct to store captured data.
  - Has an implicit call method defined
- Syntax
  - Complete type cannot be written explicitly   

---

### Tuples versus grouping

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
    f();
    g();
    h();
    i();
}
```

#### Short Answer

The output will be 123

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

Lifetime elision rules are different so the following does not compile 

```rust
fn main() {
    let closure = |x: &i32| x; // ❌
}
```

Beause it expands to

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

How are they implemented? https://huonw.github.io/blog/2015/05/finding-closure-in-rust/

- the body block of the closure is analyzed
- variables in the body that refer to variables in the surrounding scope are Marked as captured
- struct generated at compile time with as fields the references to the captured variables, it serves as the environment for the closure body
  - the struct is invisible and out of reach for the programmer in normal Rust code
  - This makes closures part of the family of unnameable types (also called Voldemort types)

Disadvantages
- The exact type of a closure struct cannot be written out to type an input or output argument 

Solution: use a trait object

```rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

---

### Move

You can change the default behaviour to only create references to the surrounding environment. You can also move variables inside the struct representing the environment of the closure. This is done by using the move keyword.
- takes ownership of variables in the surrounding scope
- inside the closure the variables are used by reference
  - does not make the closure FnOnce in itself

---
## Variants of closures

There are different kinds of closures based on the signature of their call method on the underlying struct:
- Fn:
    - the signature is call(&self),
    - the body of the closure only may have
        - immutable references to its containing scope
        - values that were moved from the containing scope (and only use them by reference afterwards)
    - can be called from anywhere, multiple times
    - Must implement FnMut

---

- FnMut:
    - the signature is (& mut self)
    - the closure can have
        - mutable references to its containing scope
        - immutable references to its containing scope
    - cannot consume or move out captured variables
    - can be called more than once, but only once at a time, must implement FnOnce

---

- FnOnce
    - signature is call(self)
    - can only be called once
    - Can move variables that are moved in out
        - Other words:
            - consume captured variables
            - apply functions to them and call by value, not by reference
        - This means that it is not Fn or FnMut, because those should be able to be called multiple times
    - and mutate,
    - implemented by every closure
  - Only implement copy clone send and sync when their contents do

---

### Interpretation of the return keyword

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

#### Short answer

This will output 2. Why?

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

https://dtolnay.github.io/rust-quiz/33

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

fn main() {
    (|| .. .method())();
}
```

#### Short answer

In this case main would print 24.

#### Long answer

This code is parsed as 

1. `(|| ..).method()`
2. this an invocation of our impl of Trait on `FnOnce() -> T` where T is inferred to be RangeFull. 
---
## Iterators


### Into iter vs. iter

#### Question

What is the difference between `into_iter` and `iter`?

#### Answer

The iterator returned by into_iter may yield any of T, &T or &mut T, depending on the context.

The iterator returned by iter will yield &T, by convention.

---

### Lazy map

https://dtolnay.github.io/rust-quiz/26

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

#### Short answer
```
112031
```

#### Long answer

The closure provided as an argument to map is only invoked as values are consumed from the resulting iterator. The closure is not applied eagerly to the entire input stream up front.---

---

## Type elision / erasure

Sometimes
- we don't know the type
  - the particular instance of a trait we exactly need as input or output for a function signature
  - the actual type is hidden from the user. These types are called unnameable or Voldemort types
- we know the type, but the full type is too long to be readable
  - iterator implementors
  - future combinators (see next session)


solution:
- For local variables 
  - use the wildcard `_`
- In type declarations for functions, traits or structs, 
  - the type cannot be inferred by the compiler
  - use opaque types

   
Benefits of opaque types
- hide specific types,
- cleaner API
- underlying concrete type can be an unnameable type
   
---

### Trait objects

First type of opaque types is a **trait object**:

- synonyms
  - dynamic dispatch
- syntax
  - Marked with the dyn keyword
- semantics
  - a pointer to an object
  - a pointer to a method table, called a **vtable** with function pointers for each method of the trait
- Disadvantages
  - Cannot be pushed on the stack directly, 
  - has to be on the heap or behind a pointer on the stack
  - Method call is determined at runtime, less optimizations
- How can it be created? Which traits can be used to create trait objects? The ones that are object safe.
    - cannot have generic methods
    - The return type isn't Self.


---

### Dynamic versus static dispatch

#### Question
What is the output of this Rust program?
```rust
trait Base {
    fn method(&self) {
        print!("1");
    }
}

trait Derived: Base {
    fn method(&self) {
        print!("2");
    }
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

fn main() {
    dynamic_dispatch(&BothTraits);
    static_dispatch(BothTraits);
}
```

#### Long answer 

- Dynamic dispatch: The forwarding is done by reading from a table of function pointers contained within the trait object. Expanded to `<dyn Base as Base>::method`. Since the argument was obtained by converting a BothTraits to dyn Base, the automatically generated implementation detail will wind up forwarding to `<BothTraits as Base>::method` which prints 1.
- Static dispatch: Type inference within generic functions in Rust happens independently of any concrete instantiation of the generic function i.e. before we know what T may be, other than the fact that it implements Base. By the time that T is decided, it has already been determined that x.method() is calling `<T as Base>::method` or `<BothTraits as Base>::method` which prints 1.

---

### Lifetimes for trait objects

a trait object's lifetime bound is inferred from context

Examples

```rust
trait Trait {}

// elided
type T1 = Box<dyn Trait>;
// expanded, Box<T> has no lifetime bound on T, so inferred as 'static
type T2 = Box<dyn Trait + 'static>;

// elided
impl dyn Trait {}
// expanded
impl dyn Trait + 'static {}

// elided
type T3<'a> = &'a dyn Trait;
// expanded, &'a T requires T: 'a, so inferred as 'a
type T4<'a> = &'a (dyn Trait + 'a);


```


---

```rust
use std::cell::Ref;

// elided
type T5<'a> = Ref<'a, dyn Trait>;
// expanded, Ref<'a, T> requires T: 'a, so inferred as 'a
type T6<'a> = Ref<'a, dyn Trait + 'a>;

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

https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#6-boxed-trait-objects-dont-have-lifetimes

---

### DSTs

Trait objects are part of the family of dynamically sized types (DST) 

Synonyms
- unsized types
Syntax
- `GenericType<D: ?Sized>`
Semantics
- Types that don't fit on the stack as local variables by value



---

### Examples of DSTs

Examples
- slices: fat pointer has address and length  
  - str
- trait objects:
  - dyn Trait

Related concepts:
- References to DSTs are called **fat pointers.** https://stackoverflow.com/questions/57754901/what-is-a-fat-pointer

Counterexamples? Most types are not DSTs because their size is known at compile time.
- They implement the trait Sized
- DSTs don't implement Sized


---

### Impl trait

The other type of opaque types is impl Trait.

Synonyms
- anonymous types

syntax
- written `impl Trait` 
- syntax sugar for hardcoding a specific type that can be inferred by the compiler
semantics
- No extra heap allocation
- No dynamic dispatch overhead

Implementation:
- "return position impl Trait" (RPIT) in Trait definitions (allowed since dec 2023)
  - the anonymous type in the impl T return type is a kind of associated type

---

#### `impl` everywhere

There is an effort underway to make `impl` type elision possible everywhere

TODO

---
## New lifetime capture rules

- new lifetime capturing rules for return position impl traits


Rust 2021: the following compiles because `impl` does not capture `s`
```rust
fn indices<'s, T>(
    slice: &'s [T],
) -> impl Iterator<Item = usize> {
    0 .. slice.len()
}
```

I Rust 2024: `impl` will capture lifetime parameter `'s`.
```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let i = indices(&data);
    data.push(4); // <-- Error!
    i.next(); // <-- assumed to access `&data`
}
```

---

- use bound": `impl Trait + use<'x, T>`, for example, would indicate that 
  - the hidden type is allowed to use `'x` and `T`
  - but not any other generic parameters in scope

To exempt from lifetime parameter capture, use 

```rust
fn indices<'s, T>(
    slice: &'s [T],
) -> impl Iterator<Item = usize> + use<> {
    //                             -----
    //             Return type does not use `'s` or `T`
    0 .. slice.len()
}
```

Advantages

- explicit capturing of specific or none of the lifetimes in the arguments
- consequences: 
  - implementation can be changed 
  - but signature does not have to be changed




See  https://blog.rust-lang.org/2024/09/05/impl-trait-capture-rules.html