# Notes
- https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-10.html#%_sec_1.1.6

## Lisp
Programming language invented in the late 50s. Created to reason about the use of recursion equations.

Created by John McCathy and based on his paper "Recursive Functions of Symbolic Expressions and Their Computation by Machine" to solve programming problems such as the symbolic differentiation and integration of algebraic expressions. It included for this purpose new data objects known as atoms and lists.

The first Lisp interpreter was implemented by McCarthy with the help of colleagues and students in the Artificial Intelligence Group.

Lisp = LISt Processing

## Pretty printing
Formatting convention in which each long combination is written so that the operands are aligned vertically. The resulting indentations display clearly the structure of the expression.

## Normal-order evaluation
Fully expand and then reduce.

## Applicative-order evaluation
Evaluate the arguments and then apply.

## AOF
Ahead of time - before it is needed.

## JIT
Just in time - right before it is needed.

## Tree accumulation
A way to recursively get all sub values to form the final value.

## Programming language
A language used to instruct computer what to do.

Programming languages are compiled into byte code. They can be compiled AOF or JIT.

Any powerful programming language should be able to describe primitive data and primitive procedures and should have methods for combining and abstracting procedures and data

## Predicate
An expression whose value is interpreted as either true or false.

## Declarative programming
"What is" descriptions.

## Imperative programming
"How to" descriptions.

## Procedural abstraction
Black boxes until we open them. Abstraction of the procedure implementation. The importantce of knowing the implementation of the procedure is deferred to later when we need it.

Names should be local and should not matter to the user. They should be encapsulated (bound to) in the abstraction.