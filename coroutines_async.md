---
title: Async Rust
sub_title: Coroutines and asynchronous functions
author: Willem Vanhulle
options:
  end_slide_shorthand: true
---



## Co-operative vs pre-emptive multitasking

When a CPU has to accomodate for multiple threads (or more general tasks), it needs to have a way to switch between different tasks. This is called **context switching**.

- cooperative multitasking
  - process themselves say when they can't make progress
  - in practice context switching happens on yield points
- pre-emptive multitasking
  - context switching happens by force on arbitrary points 
  - could be useful when dealing with listeners in the UI for responsiveness if no yield points

See https://stackoverflow.com/questions/46015648/cooperative-scheduling-vs-preemptive-scheduling


## Coroutines

Synonyms:
- Generators

Semantics
- has a signature like a function and a closure
- can capture variables by reference or by value like closure
- can be suspended during execution
- can be resumed before final return


Syntax
- How can I write a coroutine

```rust
#![feature(coroutines, coroutine_trait, stmt_expr_attributes)]

use std::ops::Coroutine;
use std::pin::Pin;

fn main() {
    let mut coroutine = #[coroutine] || {
        println!("2");
        yield;
        println!("4");
    };

    println!("1");
    Pin::new(&mut coroutine).resume(());
    println!("3");
    Pin::new(&mut coroutine).resume(());
    println!("5");
}
```

A **generator** is a simple kind of coroutine:
- Used to generate values for a list
- does not receive information when resumed. Coroutines in general may receive information when they are resumed.
- not tied to an event loop

Advantages
- Can be used to write concurrent code
  -  more efficient code that is cooperative, yields control to the caller if an external event did not happen
- Can be used to write iterators

Benefits
- expensive threads can be replaced by coroutines: 
  - smaller and more efficient context switching
- might be an easier way to write iterators
- can be used to implement async
- Optimizing utilizing computational resources (registers, caches, cores, ...)
- Not necessarily but possibly parallel

See
- https://doc.rust-lang.org/beta/unstable-book/language-features/coroutines.html

## Types of coroutines

There are to types of implementations for coroutines:
- stackful
  - as functions are called as part of the coroutine, their frames are pushed on the stack
  - each coroutine has its own stack
  - more control over the execution state 
    - easier to handle deeply nested function calls
  - more memory heavy
    - context switching is slower
- stackless
  
  - only store resumption point in an enum with necessary variables
  - more memory efficient and faster
  - more complex to implement for nested calls
  - impossible to display stack in a debugger

See
- https://langdev.stackexchange.com/questions/697/what-are-the-benefits-of-stackful-vs-stackless-coroutines
- https://without.boats/blog/why-async-rust/

## Stackless coroutines

- How are they implemented in Rust? 

```rust
pub trait Coroutine<R = ()> {
    type Yield;
    type Return;
    fn resume(self: Pin<&mut Self>, resume: R) -> CoroutineState<Self::Yield, Self::Return>;
}
```
The `Coroutine` trait is implemented on an enum.
Enum variants represent states of a state machines.
At every yield, the captured variables have to be referenced.

```rust
enum __Coroutine {
    Start(&'static str),
    Yield1(&'static str),
    Done,
}
```

The resume method progresses the states of the state machine:

```rust
impl Coroutine for __Coroutine {
    type Yield = i32;
    type Return = &'static str;

    fn resume(mut self: Pin<&mut Self>, resume: ()) -> CoroutineState<i32, &'static str> {
        use std::mem;
        match mem::replace(&mut *self, __Coroutine::Done) {
            __Coroutine::Start(s) => {
                *self = __Coroutine::Yield1(s);
                CoroutineState::Yielded(1)
            }

            __Coroutine::Yield1(s) => {
                *self = __Coroutine::Done;
                CoroutineState::Complete(s)
            }

            __Coroutine::Done => {
                panic!("coroutine resumed after completion")
            }
        }
    }
}
```



Disadvantage  
- unstable Rust language feature

See
- https://doc.rust-lang.org/beta/unstable-book/language-features/coroutines.html#coroutines-as-state-machines
- https://tmandry.gitlab.io/blog/posts/optimizing-await-1/
  
## Pinning

In general the enum `__Coroutine` might contain self-references. 

To make progress in the coroutine, we need to make sure the coroutine is not moved, otherwise the self-reference may become invalid.

So we need to pin the coroutine, since it contains references and shouldn't be moved. 

```rust
Pin::new(&mut coroutine).resume(());
Pin::new(&mut coroutine).resume(());
```

Managed automatically by run-times such as Tokio.
## Async-await

Semantics
- A computational model to deal efficiently with external events
  
Syntax  

```rust
use zenoh::prelude::r#async::*;

#[tokio::main]
async fn main() {
    let session = zenoh::open(config::default()).res().await.unwrap();
    session.put("key/expression", "value").res().await.unwrap();
    session.close().res().await.unwrap();
}
```

Advantages
- All the advantages from stackless coroutines such a more efficient multit-tasking
- Respond efficiently to network events

Disadvantages
- Cannot be mixed easily with synchronous code, colors functions
- Bounds have to be added explicitly to a desugared function signature using `impl Future`
 
See
- https://docs.rs/zenoh/latest/zenoh/

## Common async code patterns 

- How to create an asynchronous iterator?

See
- https://blog.yoshuawuyts.com/async-iterator-trait/
- https://github.com/Kobzol/async-iterator-examples/tree/main

### Async implementations

- `async fn` function declarations desugar to `impl Future` (RPIT)
- every `Future` is a stackless coroutine
- The `impl Future` captures all the lifetime variables, unless opted out

- Similarly to coroutines, inside a future
  - Local references or variables which are valid across await points are stored in a struct
  - A future can be send only if the internal data is send.

## Async in traits

In traits, you might encounter the Rust compiler message:

```
use of async fn in public traits is discouraged as auto trait bounds cannot be specified
```

without any Send bound, you'll have issues in using this trait (generically) in contexts where you need to run the async function in a multi-threaded runtime and/or spawn it as a task in such a runtime

https://users.rust-lang.org/t/async-in-public-trait/108400
## Aynchronous functions as coroutines

If every function was a couroutine, you could put the different concepts in a table.

The following diagram comes from https://without.boats/blog/coroutines-async-and-iter/


|                | YIELDS               | RETURNS         | RESUMES         |
|----------------|----------------------|-----------------|-----------------|
|                |                      |                 |                 |
| BASE CASE      | `!`                  | `Self::Output`  | `()`            |
|                |                      |                 |                 |
| FUTURE         | `()`                 | `Self::Output`  | `&mut Context`  |
|                |                      |                 |                 |
| ITERATOR       | `Self::Item`         | `()`            | `()`            |
|                |                      |                 |                 |
| ASYNCITERATOR  | `Poll<Self::Item>`   | `()`            | `&mut Context`  |
|                |                      |                 |                 |


Explanation of table

- normal functions: considered as coroutines, don't yield, so the never type is used for yield. They resume type is only relevant for coroutines that yield, so it is empty for normal functions.
- futures: When a future is viewed as the state of a call to a coroutine, it's poll function is the function that resumes the coroutine. But to poll a future, you need pass the context which contains the waker. The waker is way to notify the async runtime, in case it is currently pending, that in the future the future is ready to be polled again.

## History of async

- 2009: Mozilla sponsors Rust
- 2013: Green threads are threads managed by a runtime in user-space and can be pre-emptive or cooperative. They make it possible to continue executing multiple tasks in parallel. 
- 2015: native threads: better for compatibility, less overhead
- 2016: Tokio, `Future` trait
- 2018: async/await syntax

[why-async-rust](https://without.boats/blog/why-async-rust/)

## Implementation of async runtimes

- How is an asynchronous runtime implemented?
  - Event loop 
  - switch tasks
  
See for a mini implementation example https://tokio.rs/tokio/tutorial/async

- How many threads does an async runtime manage?
  - As many as there are CPU cores.
- Which asynchronous runtime should I use?


## Common issues in async applications

- When does it make sense to store pending futures?
  - You should leave this task to the runtime.
- Do lifetimes follow the same rules in async code?
  - Yes, but async function have to be desugared and the resulting opaque output future should be given the right lifetime bounds 

## Cutting edge

- Async closures https://blog.rust-lang.org/inside-rust/2024/08/09/async-closures-call-for-testing.html
- async drop