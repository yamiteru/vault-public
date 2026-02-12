Collection of homogenous threads that are each capable of carrying out CPU intensive tasks.

For example in NodeJS the underlying library `libuv` uses 4 threads in a pool for performing low-level I/O operations.

Usually the pool size won't need to dynamicaly change throughout the lifetime of an application. In most cases we work with a thread pool with a fixed size.

No single thread should get too much work to handle and no threads should be sitting there idle without work to do.

## Strategies

### Round robin
Each task is given to the next worker in the pool, wrapping around to the beginning once the end has been hit. So, with a pool size of three, the first task goes to Worker 1, then Worker 2, then Worker 3, then back to Worker 1, and so on.

### Random
Each task is assigned to a random worker in the pool. Although this is the simplest to build, being entirely stateless, it can also mean that some of the workers are sometimes given too much work to perform, and others will sometimes be given too little work to perform.

### Least busy
A count of the number of tasks being performed by each worker is maintained, and when a new task comes along it is given to the least busy worker. This can even be extrapolated so that each worker only has a single task to perform at a time.