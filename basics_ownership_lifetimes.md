---
title: Fundamentals of Rust
sub_title: Types, ownership and lifetimes
author: Willem Vanhulle
options:
  end_slide_shorthand: true
---

## Types

There are also some more exotic types in Rust:

- General algebraic datatypes (GADTs)?
  - product types: tuples, structs
  - sum types: enums
  - combinations: 
    - enums with variants being structs
    - structs with fields being enums

## Categorizations of types

There are two kinds of types:

- types with values that have an upper bound on the size: 
  - go on the stack
  - fixed size on stack
  - operations can be easier optimized
- types with values that can grow: 
  - go on the heap
  - cost more time

Avoid all heap allocations as much as possible.

### Ranges and tuples

#### Question

```rust
fn main() {
    let (.., x, y) = (0, 1, ..);
    print!("{}", b"066"[y][x]);
}
```

#### Short answer

```
54
```

#### Long answer

1. On the right-hand side (0, 1, ..) is a tuple with three elements, the third one having type RangeFull
2. On the left-hand side matches a tuple with 2 or more elements, binding the second-last one to x and the last one to y.
3. `b"066"[..][1]`
4. the byte b'6' of type u8. When printed, we see the decimal representation of the byte value of the ASCII digit 6, which is the number 54.

## Basic operators

### Prefix decrement

#### Question

What is the output of the following program?

```rust
fn main() {
    let mut x = 4;
    --x;
    print!("{}{}", --x, --x);
}
```

#### Short answer

```
4
```



#### Detailed answer

Rust does not have a unary increment or decrement operator.


### Postfix decrement

```rust
fn main() {
    let mut a = 5;
    let mut b = 3;
    print!("{}", a-- - --b);
}
```

#### Short answer

#### Long answer

`a - (-(-(-(-b))))`

## Scope allocation

Semantics
- Local variable assignments allocate data on the stack
- The data remains allocated until the end of the scope
- At the end of the scope of the variable, data is deallocated

Syntax
- A scope is delimited by curly braces

Examples
- code that locks a mutex essentially includes the logic that the lock will be released when execution leaves the scope of the object

Synonyms
- With blocks in Python and file handles
- in C++: Resource Acquisition Is Initialization (RAII)

## Destructors

Called when allocation is deallocated.

### Infallible matching and destructors

#### Question
What is the output of this Rust program?

```rust
use std::fmt::{self, Display};

struct S;

impl Display for S {
    fn fmt(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("1")
    }
}

impl Drop for S {
    fn drop(&mut self) {
        print!("2");
    }
}

fn f() -> S {
    S
}

fn main() {
    let S = f();
    print!("{}", S);
}
```

#### Answer

```
212
```

#### Detailed answer

No value of type S gets dropped within the body of function f. 

1. The function f conjures an S and returns ownership of it to the caller of f; the caller determines when to drop the S of which it received ownership.
2. The S in let S = f() is a unit struct pattern (not a variable name) that matches a value of type S via destructuring but does not bind the value to any variable. 
3. As no variables are declared on this line, there is no variable that could be the owner of the S returned by f() so that S is dropped at that point, printing 2. 
4. The second line of main conjures a new S, prints it, and drops it at the semicolon.

## Ownership


Semantics
- a way to manage finite resources such as the stack and the heap
- an extension of scope based resource management

Syntax:
- Regions in code with variables that own resources

Synonyms:
- Object lifetime
- Region based memory allocation


## Encoding ownership in types

inside blocks and expressions, there are two categories of variables, members and fields:
- owning types: 
  - types that are owned by the current block
  - Owning variables are bound to a scope
  - may pass ownership to a different scope
  - as long as they have the ownership, they are responsible for destroying or de-initializing the data they refer to a the end of the scope
  - cannot be duplicated to avoid resource usage
- non-owning types: types that are not owned by the current block
  - can be more easily duplicated because they do not use as much resources.


## Ownership as a tree


- ownership is also applied to nested data structures
- The owner can be moved without moving his children

Syntax:
- Invisible when created
- Visible in the error messages 
  - when trying to use a variable that has been moved

Consequences:
- Every piece of data in Rust has an owner
- The root owner is a code block
- Ownership relationship forms disjoint trees


## Root ownership 

Semantics
- The roots of the ownership tree are locally declared variables inside blocks. 
- Declared local variables are owned by the surrounding block. 
- Local variables are the top-most "owned" type

Implementation
- on the stack

Syntax:
- With `let`: locally declared variables

Examples:
- arrays
- numbers
- characters

### Constructors and drop

#### Question

```rust
struct S;

impl Drop for S {
    fn drop(&mut self) {
        print!("1");
    }
}

fn main() {
    let s = S;
    let _ = s;
    print!("2");
}
```

#### Answer

output is 

```
21
```

#### Detailed answer

The pattern `let _ = s` is a special pattern that does not move or consume `s`. `_` is a way to ignore a value without triggering a move.


## Ownership indirection

Semantics
- local variables can own data on the heap
- data on the heap can own other data on the heap
- gives a level of indirection, 
- ownership relationships between children of the ownership tree

Examples
- A vector points to and owns a buffer on the heap, it indirectly owns the value at each index.
- A struct owns the values of each field
- An enum variant owns the values of each field
- **smart pointers**: container types that own data that is allocated on the heap.
  - A `Box` 
  - A more complicated version is  `RefCell`, which enables some kind of locking so that no more than variable can modify the content at the same time. 
  - The multi-threaded version of `RefCell` is `Mutex`. A mutex is a programming concept that is frequently used to solve multi-threading problems. It is a synchronization primitive that can be used to protect shared data from being simultaneously accessed by multiple threads.
  
Related:
- C also has owning smart pointers such as mutexes.

## Sharing ownership

By default all variables have a single owner. If you want, you can create data that has multiple owners through `Rc` and `Arc`
Instances of this type can be cloned, when no instances remain, the reference count becomes zero and the data is de-allocated. This is called *reference counting*. Reference counting is the default way to share data in standard garbage-collected dynamic languages.

### Size of shared references

https://dtolnay.github.io/rust-quiz/30

#### Question

```rust
use std::rc::Rc;

struct A;

fn p<X>(x: X) {
    match std::mem::size_of::<X>() {
        0 => print!("0"),
        _ => print!("1"),
    }
}

fn main() {
    let a = &A;
    p(a);
    p(a.clone());
    
    let b = &();
    p(b);
    p(b.clone());
    
    let c = Rc::new(());
    p(Rc::clone(&c));
    p(c.clone());
}
```

#### Short answer

111011

## Moving ownership

Semantics:
- Transfer the ownership of allocated data to another place in the program
- The original location of the ownership parent is deallocated

Syntax:
- Invisible

Implementation
- Moving data that is owner of some other data will not move all its children in memory
- Only a shallow copy is made

Examples:
- Moving through assignment
- Moving through assignment of blocks: transfers ownership of the variable associated with the last expression to the outer block
- Moving through capturing
  - Capture by constructors
  - Capture by functions

Related:
- In C++ a deep copy is made when assigning owning expressions




## References


Synonyms
- a *borrow*. Using references in Rust is called *borrowing* data. 

Semantics
- An alias of a piece of data that does not capture ownership, not responsible for cleaning up
- Can be dropped manually or at the end of the scope. 

Implementation:
- an address on the stack (local reference variables) or the heap (smart pointers)
- may contain extra data such as length or size 
- The address of a reference can be printed with `println!("{:p}", &1);`.

Advantages:
- You can share data without copying, so they can increase the speed of your program.

## Referents

The referent is the data at the location pointed to by a reference. We say that the referent is **being borrowed** by its references.

Referents cannot be moved out. 


## Dereferencing 

Semantics
A reference in Rust is any variable that implements the `Deref` trait (which is a kind of type class or interface). This means that it needs to have a *dereferencing* operation implemented.

Syntax
The dereference operator `*` can appear on two sides of an assignment:

- On the left of an assignment 
  - can be used to update the value pointed to by a mutable reference if that value is declared mutable
- On the right of an assignent: accesses the value behind a reference
  - for references that point to copy types it will create a copy and assignment to the left 
    - types such as integers
  - for references that point to non-copy types, it will create a compile time error, since the ownership can not be moved through a reference

## Lifetimes

Syntax:
- `'a`

Semantics:
- The lifetime of a reference. The logical timeframe within your program that a reference is valid. 
- Cannot shrink or grow

Usage:
- Lifetimes can be elided or omitted? See https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md
- When elided, the compiler will choose the shortest lifetime based on all the code paths.
- Should be given meaningful names

Quiz questions:
- What is the meaning of `&'static T`?

Related:
- Which other programming languages support lifetimes?


## Immutable references 

Syntax: `&`

Semantics:
- make an alias for existing immutable data
- allows creation of multiple immutable references to the same data
- all data owned by the aliased referent becomes read-only

Advantages
- allows for sharing of data in a larger programs between different components. 



https://dtolnay.github.io/rust-quiz/6
```rust
use std::mem;

fn main() {
    let a;
    let a = a = true;
    print!("{}", mem::size_of_val(&a));
}
```

```rust
let a;
let b = a = true;
print!("{}", mem::size_of_val(&b));
```
The value of an assignment is `()`, so we can rewrite the program as:

```rust
let a = true;
let b = ();
print!("{}", mem::size_of_val(&b));
```

The `size_of_val` gives the size of the referent.



## Mutable references

Semantics
- Aliases mutable data 
- Data can be modified through the created reference only
- Data that is owned by the referent can only be accessed through the mutable reference
- owning parents of the referent cannot be accessed

Syntax:
- `let mutable_reference = &mut v`


### Reference types

If you see a type parameter `T` then you know that 
- `T` is a super set of `&T` and `&mut T`
- `&T` is disjoint from `&mut T`


```rust
trait Trait {}
impl<T> Trait for T {}
impl<T> Trait for &T {} // ❌
impl<T> Trait for &mut T {} // ❌
```

## Pointers


Raw pointers in Rust are like references but do not have a lifetime or rules around borrowing.

Semantics
- a kind of reference that may point to invalid memory
  
Syntax
- mutable: `*mut`
- immutable: `*const`.
  
Disadvantages:
- Dereferencing of raw pointers has to happen in an unsafe environment

See:
- [pointers vs. references in C](https://stackoverflow.com/questions/4995899/difference-between-pointer-and-reference-in-c)
- [pointers vs. references in Rust](https://stackoverflow.com/questions/62232753/what-are-the-differences-between-a-pointer-and-a-reference-in-rust)

## Structs and fields

Synonyms
- classes
- objects
- records

Syntax
- `Struct`

Semantics
- defines a new type
- A collection of related fields for one entity

## Referencing fields

In general for a given struct `struct`, the notation `struct.field` will return a reference to the value of the field `field` of the struct. 
Implicitly, the `struct.` access is replaced by a `(*struct).`.

So the following are equivalent:

```rust
fn main() {
    let my_box = MyStruct { field: 42 };
    // Implicit dereferencing
    println!("{}", my_box.field); // This will also print 42
    // Explicitly dereferencing
    println!("{}", (*my_box).field); // This will also print 42
}
```

You can assign a reference to a struct and a field access to a variable. In that case, the struct is borrowed and it can still be used afterwards
 
## Assigning fields

You can also assign a field directly to a variable
- the field is copy: the field is copied to the variable
- the field is not copy: the struct is partially moved and
  - the struct becomes unusable after the assignment because it is then empty


If you assign a nonfield to a variable without creating a reference, you will move outside of the struct

```rust
fn main() {
    let my_struct = MyStruct { field: String::from("Hello") };

    // Accessing the field directly (returns a reference to the field)
    let field_ref: &String = &my_struct.field; // No move occurs here
    println!("{}", field_ref); // Prints: Hello

    // Trying to explicitly move the field out (will work)
    let moved_field = my_struct.field; // This will work and move the String
    println!("{}", moved_field); // Prints: Hello

    // At this point, `my_struct` cannot be used to access `field` anymore
    // println!("{}", my_struct.field); // This will cause a compile-time error
}
```
