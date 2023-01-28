A simple functional reactive library with built-in data validation and TypeScript inference.

Managing ever changing states can be hard. You can very easily shoot yourself in a foot. Usually people use complex libraries built on top of complex principles like Redux or Mobx.

Functional reactive programming is declarative. This means that you don't concern yourself with the "how" but "what". In other words you just define relationships between reactive values and the library takes care of the rest.

Reval and its values can be easily extended since it uses [[μEve]] for reactivity, [[Vaal]] for data validation and [[Pipem]] for declaring value behavior.