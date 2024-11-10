---
title: Advanced Rust, part 2
sub_title: Closures and type elision
author: Willem Vanhulle
options:
  end_slide_shorthand: true

---

## Contents

Mix of information and quizes (from Tolnay) about:

1. Function traits
2. Closures
3. Iterators
4. `impl Trait`
5. Trait objects


---

## Functions in Rust

There are different kind of functions in Rust:

- **function items**
- **function pointers**
- **closures** (functions in other languages)
- **blocks** (zero-argument closures)

From most applicable to least applicable
- `Fn`: may be called infinitely 
- `FnMut`: may mutate variables
- `FnOnce`: can only be called once

Relationship: `Fn => FnMut => FnOnce`

---

## Function items

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->



The simplest kind of function.

They appear as:
- free functions `fn foo() {}`
- methods of a type, struct or enum



Function are references to immutable code in compiled binary
- can easily be cloned or copied  
- can be optimized away `=>` no pointer needed `=>` zero sized


Functions implement all the function traits.

<!-- column: 1 -->


<!-- pause -->


The compiler combines the signature of a function item with the location of its body into a unique type. 

The type of a function item encodes the information of what function will be called. 


### Generic function items

In the case of generic functions, 

- every distinct instantiation with a generic type calls a separate mono-morphized version of the function item. 
- more type-specific optimizations possible

---


## Function pointers
<!-- column_layout: [1, 1] -->

<!-- column: 0 -->



Like a function item, but can be reassigned at run-time.

Function body is not necessarily known at compile-time.

```rust
fn add(x: i32, y: i32) -> i32 {
    x + y
}

let mut x = add(5,7);

type Binop = fn(i32, i32) -> i32;
let bo: Binop = add;
x = bo(5,7);
```

The type of a function pointer is declared with `fn()`

But it is 
- a primitive type
- not a trait, so cannot be used as a trait bound.

<!-- column: 1 -->

<!-- pause -->


How do you get a function pointer?

- As an alias for a function item
- Cast a non-capturing closure (see further)


More properties of function pointers:

- Implements all function traits `Fn`, `FnMut`, and `FnOnce`
- Thread-safe, `Sync` and `Send`, since they don't capture anything
- Implemented as a pointer:
  - Slower than calling a function item or closure directly
  - Faster then a dynamic dispatch with  `dyn Fn`


Every function pointer
- Can be passed as argument to
  - function items
  - other function pointers
- Can be returned from a function item


Useful in FFI with languages that don't support closures

---

## Function pointers as associated items

Function pointers can also be attached to traits

```rust
/// Trait for transorming a type into a state machine.
pub trait IntoStateMachine
where
    Self: Sized,
{
    ...
    type State;
    const ON_TRANSITION: fn(&mut Self, &Self::State, &Self::State) = |_, _, _| {};
}
```
See [statig](https://github.com/mdeloof/statig/blob/main/statig/src/into_state_machine.rs)

Function pointers cannot be async unless you allocate a `Pin<Box<F>>` on the heap.


---


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

<!-- column: 1 -->

### Short answer

0: Why?

<!-- pause -->

### Long answer

1. The function `size_of_val` returns the size of the value a pointer points to. 
2. `&f` points to a function item. 
3. Function items do not occupy data, they only occupy space as code.

---


<!-- column_layout: [1, 1] -->

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

### Short answer

```
8 (64-bit)
4 (32-bit)
```

<!-- column: 1 -->


Why?

<!-- pause -->

### Long answer

1. `my_function` is a function item.
2. During assignment it is cast as a function pointer.
3. Function pointers have the same size as every other pointer.
4. On a 64 bit processor, a pointer is 8 bytes = 8 * 8 bits long.

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

1. For the first statement, 
   1. we start with the function item  `a::<u8>`
   2. we cast the function item to a function pointer
   3. the parameter of `f` now has type `fn(fn(u8))` 
   4. We call `d` which computes the size of `fn(fn(u8))` but it is a function pointer, so its size would be 8 on a system with 64-bit function pointers
2. In the second statement,
   1. We start with the same function item `a::<u8>`.
   2. Now we pass `a` without casting as a function pointer
   3. `d` is directly called with `T` being the function item type  `a::<u8>`, 
   4. Function item types are zero-sized.


[See](https://dtolnay.github.io/rust-quiz/34)

---

## Closures

A kind of function that may **capture references** to variables in the containing scope.

Also called
- Lambda-functions
- Anonymous functions


Non-capturing closures can be converted into function pointers automatically.

Notice in the previous example from Statig we cast the trivial closure as a pointer:

```rust
const ON_TRANSITION: fn(&mut Self, &Self::State, &Self::State) = |_, _, _| {};
```

Async closures are still an unstable feature. For the moment you have to create a heap-allocated `Pin<Box<dyn Future>>`.

---

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
<!-- pause -->


### Short answer

```
1
```


<!-- column: 1 -->



<!-- pause -->


### Long answer

1. We construct a closure `print2`.
2. We cast the closure as a function pointer.
3. We construct a `struct` with `S`.
4. We call the method `f` implemented for `S`, becaus `.f()` always resolves to a method. 
 
To call the function pointer stored in field `f`, we would need to write parentheses around the field access: 

```rust
fn main() {
    let print2 = || print!("2");
    (S { f: print2 }.f)();
}
```

(See Tolnay)


---
## Closures and lifetimes


<!-- column_layout: [1, 1] -->

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
#![feature(closure_lifetime_binder)]
fn main() {
    let closure = for<'a, 'b> |x: &'a i32| -> &'b i32 { x };
}
```

Two distinct lifetimes are created for the input and output type. For normal functions, only one lifetime is created.

---

## Closure implementations

<!-- column_layout: [1, 1] -->

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

### `Fn`

- the signature of the `call` function is `call(&self)`,
- the body of the closure can only have
    - immutable references to its containing scope
    - values that were moved from the containing scope (and only use them by reference afterwards)
- can be called from anywhere, multiple times
- Must implement `FnMut`

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




### Summary

- Trait bounds
- Closures
- Dynamic dispatch
- Impl traits

---

### Questions?

This presentation was made using [`presenterm`](https://github.com/mfontanini/presenterm).