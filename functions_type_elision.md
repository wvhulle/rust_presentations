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

## Function types

> Things that **produce values**.

### Traditional function types

Rust has a few function-like things:
- Function items
- Function pointers
- Closures

### What else?

<!-- pause -->

Other magical things that can produce values...

---

## Review of function traits

Functions are categorized **using the trait system**.

1. `Fn`: Functions that do not capture mutable references and may be called infinitely
2. `FnMut`: Functions that may reference mutable variables
3. `FnOnce`: Functions that can only be called once and may move or drop variables

Function items and function pointers **satisfy all these traits**.

Closures form a ladder: `Fn => FnMut => FnOnce`. First, function items.

---

## Function items

Simplest and most fundamental kind of function. Declared with `fn`.

Properties:

- no run-time cost in terms of additional memory
- implement all function traits `Fn, FnMut, FnOnce`. 

Appear as:

- **Free functions**: also called **helper functions** `fn foo<T>() {}`.
- **Methods**: defined in `impl` blocks.

---

## Constant evaluation

Some contexts require **constant evaluation** = evaluation at compile-time.

Examples: 

- **initializer** on the right-hand side of `static` or `const` variables (**easiest way** to create data with `'static` lifetime) 
- initializer expressions for arrays of a fixed size:

```rust
let a: [Option<i32>;3] = [const { None }; 3];
```

All functions called in these contexts should be marked with `const fn f(){}`.

Body of a `const` function has some constraints (for example no iterators)

```rust
pub const fn fibonacci(n: usize) -> usize {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn main() {
    const RESULT: usize = fibonacci(10);
    println!("The 10th Fibonacci number is: {}", RESULT);
}
```
[Constant evaluation](https://doc.rust-lang.org/reference/const_eval.html)

---

## Lifetime elision 

Lifetime annotations are necessary for functions that have reference types as arguments. 

In some cases they can be omitted, following **lifetime elisision rules**.

The compiler automatically assigns lifetimes during elision.

### Free functions

If there is exactly **one input lifetime position** (elided or not), that lifetime is assigned to all elided output lifetimes.

```rust
fn print(s: &str);                                      // elided
fn print<'a>(s: &'a str);                               // expanded

fn debug(lvl: usize, s: &str);                          // elided
fn debug<'a>(lvl: usize, s: &'a str);                   // expanded
```

---

## Lifetime practice

Guess the lifetimes annotated by the compiler:

```rust
fn substr(s: &str, until: usize) -> &str;     
```

<!-- pause -->

```rust
fn substr<'a>(s: &'a str, until: usize) -> &'a str;     
```

What are the lifetimes annotated to the following?

```rust
fn get_str() -> &str;                              
fn frob(s: &str, t: &str) -> &str;                  
```
<!-- pause -->
_Illegal_ / does not compile.

---

## Methods

In argument position, the word `self` is a keyword

The type of `self` can be any **concrete** type that dereferences to `Self`:

- `Self`, 
- `&Self`, 
- `&mut Self`, 

Other possible types for `self`:

- `Box<Self>`, 
- `Rc<Self>`, 
- `Arc<Self>`
- `Pin<P>`

<!-- pause -->

The last **cannot be used as the base type** of an `impl` block (foreign type).

Probably because these have different ownership semantics.

Advantages of methods:

<!-- pause -->

- Compact notation through **chaining**.
- Group functions related through **shared state** or **intermediate computation**.

---

## Lifetime elision in methods

In the following situation:

- multiple input lifetime positions, 
- but one of them is `&self` or `&mut self`, 

Then the lifetime of the reference to `self` is assigned implicitly to all elided output lifetimes.

Example:

```rust
fn get_mut(&mut self) -> &mut T;                       
fn get_mut<'a>(&'a mut self) -> &'a mut T;
```

What are the expanded life-times?

```rust
fn args<T: ToCStr>(&mut self, args: &[T]) -> &mut Command                  
```

<!-- pause -->

```rust
fn args<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command 
```

---

## Implementation function items

Part of immutable code in the compiled binary.

Consequences:

- can easily be cloned or copied  
- function calls can be optimized away

Does the following compile?

```rust
fn foo<T>(_t: T) {}
fn bar<T>(_t: T) {}
fn main() {
    let x = &mut foo::<i32>;
    *x = bar::<i32>;
}
```

<!-- pause -->

No. 

1. `foo` starts out as a function item.
2. `foo::<i32>` is assigned to `x` with a _unique type_. 
3. `bar::<i32>` is created with another _unique type_.
4. Both types are different and cannot be stored in the same variable.

What you can do is _cast_ function items as **function pointrs** (see later)

```rust
let x: &mut fn(i32) = &mut (foo::<i32> as fn(i32));
*x = foo::<i32> as fn(i32);
```

---

## Closures

---

## Closures


### What are closures?


Because they can be anonymous, closures are useful for **functional programming**.

Functional programming languages require functions as **first-class citizens**.

We have to be able to pass them around anonymously, which is why they are also called **anonymous functions**.

In lambda-calculus closures are written using a lambda, so they are also called **lambda-functions**

For example:

```
(λx. x + 1) 5 
```

A foundation for formal verification of programming languages. 

See [Plato Stanford](https://plato.stanford.edu/entries/lambda-calculus/)


---

## Function items vs. closures

|                        | **Function items** | **Closures** |
|------------------------|----------------|----------|
| _Anonymous_ | No | Yes |
| _Capture_                | No             | Optional |
| _Recursive_              | Optional       | No       |
| _Signature_              | Required       | Optional |
| _`Fn` and `FnMut` trait_ | Yes            | Optional |
| _`FnOnce` trait_         | Yes            | Yes      |

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

<!-- pause -->

```rust
fn main() {
    let data = String::from("Hello, Rust!");
    let closure = create_closure(&data);
    closure();
}

fn create_closure<'a>(data: &'a str) -> impl Fn() + 'a {
    move || {
        println!("{}", data);
    }
}
```

---

## Variants of closures

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->


We start at the bottom of the ladder with the type of closures that can **do the most** but are also the **least widely applicable**.

**Question**: What trait is associated with this type of closures?

<!-- pause -->

### The base case `FnOnce`

The least a closure should be able to do is be called once = the **base case**.

Imagine the following closure:

```rust
let mut v = vec![];
let closure = move || v.push(1);
```

**Question**: What does this compile to?

<!-- pause -->

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


**Question**: is every `move` closure `FnOnce`?

<!-- pause -->

**Answer**: No, 
- if a closure captures something by reference the reference is moved inside the implicit struct. Afterwards the call function of the struct can  be called multiple times and the body uses the captured variables by reference. The reference stays in between executions. 
- If a closure captures something by value, the body of the call function will use it by reference. If you move it, then the closure becomes `FnOnce`.

---

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

Extremely hard.

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

Hint: did you hear about the never type `!`?

<!-- pause -->

#### Long answer

1. We define a closure x `|| { (return) || true; }`
2. `(return)` is the application of a tuple constructor function that has output type `!` because it returns before the tuple is constructed
3. This implies that `(return) || true` is of type `! || true` which evaluates to `bool || true`
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

### Closures that are `FnMut`

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

A basic example is:


```rust
let mut x = 5;
{
    let mut square_x = || x *= x;
    square_x();
}
assert_eq!(x, 25);
```

This compiles to

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

<!-- column: 1 -->


The signature of the `call` method on the implicit struct is `call(&mut self)`.

The struct may have
- mutable references to its containing scope (but importantly, does **not need to**)
- immutable references to its containing scope


Compared to `FnOnce`, a `FnMut` closure:
- cannot consume or move out captured variables
- can be called more than once for sure, so it must implement `FnOnce`
- can only be called once at a time, 


---

### `Fn` function trait

The **least capable** but also the **most widely useful** closures are those with the `Fn` trait.

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

### Async closures

On Rust stable, closures cannot be `async`. You need to put a future inside a box. This means you have to do a heap allocation. Before you can call it, it has to be pinned. 

So the final output type is `Pin<Box<dyn Future>>`.

On Rust nightly, closures can be `async` with `#![feature(async_closure)]`.

[RFC](https://rust-lang.github.io/rfcs/3668-async-closures.html)
[Rust blog](https://blog.rust-lang.org/inside-rust/2024/08/09/async-closures-call-for-testing.html)


---
## Function pointers

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

A type of functions that do not have to be named and implement all traits `Fn, FnMut, FnOnce`. 

This makes them somewhat of a mix between 

- weak closures and 
- references to function items

They are a kind of **pointers to free function items**. 

The **type of a function pointer** is denoted with `fn() -> ()`. 

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

### Question

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


### Short Answer

```
0
```

Why?

<!-- pause -->

**Answer**: In this case `f` is a function item. Function items **do not have a run-time cost**, so they do **not require memory allocation**.


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

### Short Answer

```
8 (64-bit)
4 (32-bit)
```

<!-- pause -->

### Long answer

**Answer**: The function item `my_function` is casted as a function pointer because of the type declaration of `f`. Function pointers are stored in a pointer-like form which allows them to be pass around at run-time. Every pointer is the size of a CPU register.

---


### What to do with function pointers?

Can be passed as argument to other functions or returned.

Can be an **associated item** of a trait

An example from the `statig` library:

```rust
pub trait IntoStateMachine
{
    ...
    const ON_TRANSITION: fn(&mut Self, &Self::State, &Self::State) = |_, _, _| {};
}
```

Beware that you cannot use `impl` in the signature. So async functions can only be written as function pointers if you transform them in normal function pointers that allocate a trait object (see further) of a future on the heap.

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

### Lifetime notation of function pointers

The declaration of lifetime parameters is a bit different for function pointer types.

You need to declare them before the `fn` keyword with a `for<'a>` expression.

```rust
struct S {
    function_pointer: for<'a> fn(arg: &'a i32) -> &'a i32
}
```

This is similar to the way you declare a super trait that depends on a life-time parameter.

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

**Answer**: Use **type elision**.

1. Total type elision for for local variables: use the wildcard `_` for generic parameters and let the compiler infer
2. Use partial type elision in type declarations for functions, traits or structs and only declare trait bounds with **opaque types**
   1. Compile time concrete type inference: `impl Trait` (in the following slides)
   2. Run-time trait bound checking: `&dyn Trait` (later)

I won't cover total type elision or `_`-wildcards here. They are only useful in the simplest cases.

<!-- column: 1 -->

### Partial type elisision

The benefits of opaque types:
- hide internals of underlying types
- support a wider variety of types (whole families of traits)
- underlying types can have an unnameable type (for example closures, async functions)

All in all, opaque types improve code usability. 

They should be used as much as possible where appropriate.

---

### `impl Trait`

The first kind of opaque type.

When we know the underlying can be determined at compile time we use `impl Trait`. 

Syntax sugar for hardcoding a specific type that can be inferred by the compiler

Benefits:
- No extra heap allocation
- No dynamic dispatch overhead

Synonyms
- anonymous types

---

## Basic example

(Taken from the Rust documentation)

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

# Unstable `impl` stuff

---

## The  `impl` everywhere project

A project goal on the Rust roadmap.

1. Signatures of methods in traits can use `impl Trait`, also known as "return position impl Trait" (RPIT) 
   - Implemented using a kind of associated types [See](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html)
2. Type aliases can use `impl Trait`: `type I = impl Iterator<Item = u32>`. [See](https://rust-lang.github.io/impl-trait-initiative/explainer/tait.html)
3. Traits with associated types can use `impl Trait` as bounds for the associated type (in nightly Rust)
4. In local variable bindings `let x: impl Future = foo()` (planned) 

[See](https://rust-lang.github.io/impl-trait-initiative/)

In the following slides I will give a short overview of the most recent features available in the nightly compiler.

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

This used to compile but now it doesn't anymore for consistency reasons:

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let i = indices(&data);
    data.push(4); // <-- Error!
    i.next(); // <-- assumed to access `&data`
}
```

New default = the hidden types for a return-position `impl Trait` can use any generic parameter in scope.

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

## `impl` on local variables

`#![feature(impl_trait_in_bindings)]`


```rust
fn main() {
    let x: impl Trait = foo;
}
```

---

## `impl` on type aliases

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

### Quiz

**Question**: Which of the following traits can be used to create a trait object?
- A trait that is `Sized`
- A trait that is `Clone`

<!-- pause -->

**Answer**: Neither of them. `Sized` is irrelevant since every trait object `dyn T` for `T: Sized` is `!Sized` anyway. Traits that are Clone need information about the size of `Self` to allocate on the stack. This information is not present at run-time. So they both cannot be used to create a trait object.

<!-- pause -->

### Definition of object safe

There are rules that say which traits are **object safe** and can be turned into traits.

You must be able to call methods of the trait the same way for any instance of the trait.

The properties of the trait should be such that  
- the size and shape of the arguments and return value only depend on the bare trait
- they dont depend on the instance (on `Self`) or any type arguments (which will have been "forgotten" by runtime)

[See](https://www.reddit.com/r/rust/comments/kw8p5v/what_does_it_mean_to_be_an_objectsafe_trait/)


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


```rust
fn main() {
    dynamic_dispatch(&BothTraits);
    static_dispatch(BothTraits);
}
```

<!-- column: 1 -->


<!-- pause -->

#### Long answer 


- Dynamic dispatch:
  - convert the argument of type `BothTraits` to `dyn Base`
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


Summary: method resolution on both trait objects and generic types **passes through the trait bounds**.

---

## Lifetimes for trait objects

a trait object's lifetime bound is inferred from context

<!-- column_layout: [1, 1] -->
<!-- column: 0 -->


### Inside a `Box`

Let's assume we have some trait:

```rust
trait Trait {}
```

We define a Box containing a trait object.
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


### Inside `impl` blocks

```rust
impl dyn Trait {}
```
What is the lifetime bound on `dyn Trait`?

<!-- pause -->
```rust
impl dyn Trait + 'static {}
```

### References to trait objects

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

### Borrowed references

`Ref` wraps a borrowed reference to a value in a `RefCell` box.

```rust
use std::cell::Ref;

// elided
type T5<'a> = Ref<'a, dyn Trait>;
```

What does this expand to?

<!-- pause -->

`Ref<'a, T>` requires `T: 'a`, so inferred as `'a`

```rust
type T6<'a> = Ref<'a, dyn Trait + 'a>;
```

<!-- column: 1 -->

### Traits with life-times

Some more examples:


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

One proposal is to add derive macros that signify things that have a similar implementation with different effects. 

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

Personally, I don't like this proposal.

See the blog post about Rust written by [Yoshua Wuyts](https://blog.yoshuawuyts.com/extending-rusts-effect-system/)

<!-- column: 1 -->

The Rust syntax proposal is quite ugly compared to Koka.

In Koka, there are **algebraic effect handlers**. 

For example, there are several built-in effect handlers: 
- `total` means a function that terminates on all inputs.
- `div` is for functions that never terminate.
- `exn` is for functions that panic or throw.

<!-- pause -->

The following functions all compute the square of an integer.

```koka
fun square1( x : int ) : total int   { x*x }
fun square2( x : int ) : console int { println( "a not so secret side-effect" ); x*x }
fun square3( x : int ) : div int     { x * square3( x ) }
fun square4( x : int ) : exn int     { throw( "oops" ); x*x }
```

See:
- [Koka](https://koka-lang.github.io/koka/doc/index.html)
- [Effekt](https://effekt-lang.org/)


---

## The basics of iterators

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

### Looking ahead to coroutines

An iterator is not really an effect, but it could be viewed as a special case of co-routines. 

Co-routines (as seen later) are effects. Coroutines introduce the concept of yielding.

An iterator is a special case of a coroutine that yields new values each time it is called.

Iterators are a useful intermediate data structure that are **easy to reason about** and handle.

Use them **as much as you can**.

How do you start using them?
- In most languages the bodies of iterators are written with the `yield` keyword.
- Rust goes for an approach with traits and introduces `next`.

<!-- column: 1 -->

### Common iterators in Rust

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

<!-- pause --> 


#### Long answer

This code is parsed as 

1. The inner part `|| .. .method()` is parsed as `(|| ..).method()`
...
<!-- pause -->

2. The type of `|| ..` is `FnOnce() -> T` where `T` is inferred to be `RangeFull`
3. We resolve the method call `(|| ..).method()` to the method in `impl<F: FnOnce() -> T, T> Trait for F`.
4. It prints `2` and returns a closure.
5. `(|| .. .method())()` evaluates the closure and returns `4`

[See](https://dtolnay.github.io/rust-quiz/33)

---

## Implementing `IntoIter`


<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

How does a custom iterator look?

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

How do we instantiate a new iterator over a grid?

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

This implies that an expression `for x in it` where `it: IntoIterator` desugars to:

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

The question mark operator `?` introduces a new way to change the execution flow of programs.

It is a kind of **effect** since it adds something new to the **way we compute** or execute our programs, similar to coroutines.

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

See [std](https://doc.rust-lang.org/std/ops/trait.Try.html)

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

