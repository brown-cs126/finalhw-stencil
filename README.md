# Final HW: optimizations

In this homework, you'll implement some optimizations in your compiler. You'll
also come up with benchmark programs and see how well your optimizations do on a
collaboratively-developed benchmark suite.

You'll implement at least two optimizations, all of which we discussed in class:

- Constant propagation and at least one of:
- Inlining
- Common subexpression elimination

In order to make inlining and common subexpression elimination easier to
implement, you'll also write an AST pass (i.e., a function of type `program ->
program`) to make sure all variable names are globally unique.

If you're taking the class as a capstone project, you'll also write a short
document about how your optimizations work and what kind of results you end up
with.

**Due dates**:
- Benchmark programs: Wednesday, April 14 at 9pm
- Final submission: Friday, April 23 at 9pm

Because grades are due not long after the project, you cannot use late days on
this final homework.

You have some options as far as how much time and effort to put into this final
homework. If you're short on time and want to be done with the
semester--perfectly understandable!--we recommend implementing inlining and
skipping the optional extension to constant propagation. If you feel like diving
in a little deeper, implement common subexpression elimination and the optional
extension to constant propagation. It's up to you, and won't affect your grade.

## Starting code

The starting code is the same as for HW7, but without support for MLB
syntax. Lambda expressions and function pointers are not supported.

You should write all of your optimizations in the file `lib/optimize.ml`. You
can write tests in the usual way; the tester will run all of your optimizations
on every test case.

You can run the compiler with specific optimization passes enabled using the
`bin/compile.exe` executable, by passing the `-p` argument one or more
times. For instance:

```sh
dune exec bin/compile.exe -- examples/ex1.lisp output -r -p propagate-constants -p uniquify-variables -p inline
```

will execute the compiler with constant propagation, globally unique names, and
inlining enabled. You can also use this to execute an optimization more than
once--for instance, doing constant propagation, then inlining, then constant
propagation again. You can also pass `-o` instead to enable all optimizations.

## Constant propagation

Constant propagation is a crucial optimization in which as much computation as
possible is done at compile time instead of at run time. We implemented a sketch
of a simple version of constant propagation in class. Your constant propagation
implementation should support:

- Replacing the primitive operations `add1`, `sub1`, `plus`, `minus`, `eq`, and
  `lt` with their statically-determined result, when possible
- Replacing `let`-bound names with constant boolean or number values, when
  possible
- Eliminating `if` expressions where the test expression's value can be
  statically determined

**Optionally**, you can also implement re-associating binary operations
(possibly in a separate pass) to find opportunities for constant
propagation. For instance, consider the expression

```scheme
(+ 5 (+ 2 (read-num))
```

This expression won't be modified by the constant propagation algorithm
described above, but with re-association it could be optimized to

```scheme
(+ 7 (read-num))
```

## Globablly unique names

Many optimizations can benefit from a pass that ensures all names are globally
unique. Implement this pass using `gensym`. This pass should be run before
inlining and common subexpression elimination, and both of those optimizations
can then assume globally-unique names (this is an exception to the usual
principle that the order of optimizations shouldn't matter for correctness). The
`validate_passes` function in `optimize.ml` ensures that this optimization is
executed before inlining and common subexpression elimination.

## Inlining

Implement function inlining for function definitions. In general, inlining
functions can be tricky because of variable names; consider the following code:

```scheme
(define (f x y) (+ x y))

(let ((x 2))
  (let ((y 3))
    (f y x)))
```

A naive inlining implementation might result in code like this:

```scheme
(let ((x 2))
  (let ((y 3))
    (let ((x y))
      (let ((y x))
        (+ x y)))))
```

This expression, however, is not equivalent!

This problem can be solved by adding a simultaneous binding form like the one
you implemented in HW3. It can also be solved by just ensuring that all variable
and parameter names are globally unique.

You should implement a heuristic for when to inline a given function. This
heuristic should involve both (1) the number of static call sites and (2) the
size of the function body. For example, you could multiply some measure of the
size of the function body by the number of call sites and see if this exceeds
some target threshold. We recommend implementing your inliner as follows:

1. Find a function to inline. This function should satisfy your heuristics and
   be a *leaf* function: one that doesn't contian any function calls.
2. Inline static calls to the function and remove the function's definition.
3. Go back to step 1. Now that you've inlined a function, more functions may now
   be leaf functions.
   
This process will never inline recursive functions, including mutually-recursive
functions.

Please describe your heuristic in a comment in the `optimizations.ml` file.

## Common subexpression elimination

Implement common subexpression elimination. This optimization pass should find
common subexpressions, add names for those subexpressions, and replace the
subexpressions with variable references.

This optimization is more challenging to implement than inlining is. Our
suggested approach is to:

- Optimize each definition (including the top-level program body) independently. For each definition:
  - Make a list of *all* of the subexpressions in the program that don't include calls to `(read-num)` or `(print)`
  - Find any such subexpressions that occur more than once
  - Pick a new variable name for each expression that occurs more than once
  - Replace each subexpression with this variable name
  - Add a let-binding for each common subexpression
  
The most difficult part of this process is determining where to put the new
let-binding. Consider replacing the (identical) subexpressions `e1`, `e2`, and
`e3` with the variable `x`. You'll need to find the lowest common ancestor `e`
of `e1`, `e2`, and `e3`, then replace it with

```
(let ((x e1)) e)
```

In order to find this lowest common ancestor, it will likely be useful to track
the "path" to a given expression: how to get to that subexpressson from the top
level of the given definition. How exactly you do this is up to you.

## Benchmarks

There's a benchmarks repository at
https://github.com/brown-cs126/final-benchmarks. You can add your benchmarks to
that repository by [forking the
repository](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-forks)
and then [creating a pull
request](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request-from-a-fork)
adding a file to the `benchmarks` directory. As part of your grade for this
final homework, you should add *at least three* interesting benchmark programs
to this repository *by Wednesday, April 14*.

Sometime before Tuesday, April 13, the benchmarks repository will be updated with
some scripts to see how much your optimizations improve performance on the
various benchmarks.

## Capstone

If you are taking CSCI 1260 as a capstone, you should submit a short (1-2 page)
PDF document describing your implementation of these optimizations and their
effects on your compiler's performance (the benchmarking scripts may help with
this). This will serve as your capstone summary!
