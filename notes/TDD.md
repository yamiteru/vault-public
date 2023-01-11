Test driven development. We first write tests and then the implementation.

By following TDD the number of bugs should reduce drastically.

Having the bug documented as a test case ensures the bug stays fixed in the future, preventing regression.

For many developers having tests in the code is an afterthought. But everyone tests their code (usually manually which eats up a lot of time).

Testing helps with sticking to [[YAGNI]].

# Testing pyramid
```
      |  E2E  |
   | Integration |
|        Unit       |
```

Application should have a lot of unit tests, fewer integration tests and eve. fewer E2E tests.

# Difficulties with adoption

- Inexpirienced team
- Slower initial development speed
- Legacy code
- Slow tests

# When not to use TDD

It's not a silver bullet. TDD introduces a high initial cost.

- It's not useful for a [[PoC]]
- When the product owner has not defined cear requirements