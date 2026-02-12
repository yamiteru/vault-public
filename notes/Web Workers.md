Web workers allow you to spawn a new environment for executing JavaScript in. JavaScript that is executed in this way is allowed to run in a separate thread from the JavaScript that spawned it.

Since we can pass multiple messages into the worker and they can take unknown amount of time we should use some kind of RPC to keep track of what outputs should be assigned to what functions. We might use JSON-RPC exactly for that. Alternatively we might implement a Command Dispatcher Pattern.

## Types
### Dedicated Worker
### Shared Workers
Can be accessed by different browser environments, such as different windows (tabs), across iframes, and even from different web workers. 

They also have a different self within the worker, being an instance of SharedWorkerGlobalScope. A shared worker can only be accessed by JavaScript running on the same origin.

Shared workers can be used to hold a semipersistent state that is maintained when other windows connect to it.
### Service workers
It's similar to Shared Workers. It functions as a sort of proxy that sits between one or more web pages running in the browser and the server.

The difference from Shared Workers is that Service Workers can exist and run in the background even when a page isn’t necessarily still open.

It does require a web page to be opened first to install the shared worker.

The most common use is for caching. It's used for intercepting communication between the server and the browser.

They have much bigger API than Dedicated Workers and Shared Workers.

Since Service Workers are async and localStorage is blocking on read/write it's not available. But async IndexedDB is available.

Using global variables in Service Workers is unpredictable since the Workers can stop and GC the global scope.




