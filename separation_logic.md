---
title: Concurrent separation logic
sub_title: Verification of Rust code
author: Willem Vanhulle
options:
  end_slide_shorthand: true
---


## Type theory

Way to verify safety of computer programs.

Similar to logic.

Natural deduction describes inference.

## Substructural logic

a structural rule is an inference rule of a sequent calculus that does not refer to any logical connective but instead operates on the sequents directly

What are the structural rules 

- Exchange: two members on the same side may be swapped
- Weaking: a hypothesis or conclusion may be extended with additional members
- Contraction: two equal members on the same side may be replaced by a single member

## Affine type theory  

No structural rule called contraction

See [Boats](https://without.boats/blog/ownership/)

```
————————————   ————————————
 x:T |- x:T     x:T |- x:T
———————————————————————————
 x:T, x:T |- (x,x) : (T,T)
——————————————————————————— <- contraction!!
   x:T |- (x,x) : (T,T)
  ———————————————————————
   λx.(x,x) : T -> (T,T)
```
Taken from [Reddit](https://www.reddit.com/r/rust/comments/eaf5ko/what_logic_does_rusts_rules_correspond_to/)

Consequence of the lack of the rule of contraction: 
- values cannot be duplicated by default
- types that are owned cannot be consumed twice

To make owned types duplicatable you have to make them `Copy` in Rust.

## Concurrent separation logic

Modelling computations utilizing resources.

Using Hoare logic.

Intermediate
https://www.youtube.com/watch?v=1GjSfyijaxo

Advanced:

- The general framework: https://iris-project.org/tutorial-pdfs/iris-lecture-notes.pdf
- Applied to Rust: https://iris-project.org/pdfs/2024-pldi-refinedrust.pdf