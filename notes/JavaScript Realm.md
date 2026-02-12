Instance of the JavaScript environement. Each realm get its own global object. You can refer to it as `global` in NodeJS, `window` in browsers or `globalThis` in any environment.

In NodeJS new realms can be created with `vm.createContext()`.

Realms always use [[Concurrency]] to run meaning they use only one instruction pointer.

