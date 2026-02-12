---
link: https://people.eecs.berkeley.edu/~ksen/papers/implicit.pdf
---

# TLDR;

The research paper provides an in-depth study of type coercions in real-world JavaScript programs, challenging the common assumption that type coercions are rare and error-prone. It finds that coercions are prevalent and mostly harmless, with only a small percentage being potentially harmful. The paper also proposes a classification of coercions into harmless and potentially harmful, which could guide the development of safer subsets of JavaScript and future language designs.

- Out of 30 manually inspected potentially harmful code locations, 22 were correct and only one was a clear bug, suggesting a very small percentage of erroneous coercions.
- 86.13% of code locations with coercions are monomorphic, always converting the same type into the same other type, indicating potential for refactoring for improved code understandability.
- 93.79% of polymorphic code locations involve conditionals where some defined value or the undefined value is coerced to a boolean.

---

Most popular programming languages support situations where a value of one type is converted into a value of another type without any explicit cast. Such implicit type conversions, or type coercions, are a highly controversial language feature.

Proponents argue that type coercions enable writing concise code. Opponents argue that type coercions are error-prone and that they reduce the understandability of programs.

We find that coercions are widely used (in 80.42% of all function executions) and that most coercions are likely to be harmless (98.85%).

JavaScript avoids throwing runtime type mismatch exceptions as much as possible so that programs can tolerate errors as much as possible.

## Findings

- Type coercions are widely used: 80.42% of all function executions perform at least one coercion and 17.74% of all operations that may apply a coercion do apply a coercion, on average over all programs.
- In contrast to coercions, explicit type conversions are significantly less prevalent. For each explicit type conversion that occurs at runtime, there are 269 coercions.
- We classify coercions into harmless and potentially harmful coercions, and we find that 98.85% of all coercions are harmless and likely to not introduce any misbehavior.
- A small but non-negligible percentage (1.15%) of all type coercions are potentially harmful. Future language designs or restricted versions of JavaScript may want to forbid them. For today’s JavaScript programs, a runtime analysis can report these potentially harmful coercions.
- Out of 30 manually inspected potentially harmful code locations, 22 are, to the best of our knowledge, correct, and only one is a clear bug. These results suggest that the overall percentage of erroneous coercions is very small.
Most code locations with coercions are monomorphic (86.13%), i.e., they always convert the same type into the same other type, suggesting that these locations could be refactored into explicit type conversions for improved code understandability.
- Most polymorphic code locations (93.79%), i.e., locations that convert multiple different types, are conditionals where either some defined value or the undefined value is coerced to a boolean.
- JavaScript’s strict and non-strict equality checks are mostly used interchangeably, suggest- ing that refactoring non-strict equality checks into strict equality checks can significantly improve code understandability.

## Harmful coersions

The most prevalent potentially harmful coercion (by number of static occurrences) are non-strict (in)equality checks that compare two objects of different types, which is a common source of confusion.

Several prevalent kinds of potentially harmful coercions involve undefined, such as concatenating undefined with a string, which yields a string that contains "undefined", and relative operators applied to undefined and a number, which always yields false.
