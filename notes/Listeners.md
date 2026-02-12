Callbacks in [[Reactive programming]] or [[Functional Reactive Programming]] waiting to be called by [[Producers]].

## The six plagues of listeners
### Unpredictable order
In a complex network of listeners, the order in which events are received can depend on the order in which you registered the listeners, which isn’t helpful.

### Missed first event
It can be difficult to guarantee that you’ve registered your listeners before you send the first event.

### Messy state
Callbacks push your code into a traditional state-machine style, which gets messy fast.

### Threading issues
Attempting to make listeners thread-safe can lead to deadlocks, and it can be difficult to guarantee that no more callbacks will be received after deregistering a listener.

### Leaking callbacks
If you forget to deregister your listener, your program will leak memory. Listeners reverse the natural data dependency but don’t reverse the keep-alive dependency as you’d like them to.

### Accidental recursion
The order in which you update local state and notify listeners can be critical, and it’s easy to make mistakes.






