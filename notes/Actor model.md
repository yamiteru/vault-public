Programming pattern for performing [[Concurrency]].

Actors communicate with the outside world by passing messages. Otherwise they have their own isolated access to memory.

The actor model is designed to allow computations to run in a highly parallelized manner without necessarily having to worry about where the code is running or even the protocol used to implement the communication.

When the messages are first received, they sit in a message queue, sometimes referred to as a mailbox.

Actors are not allowed mutate shared memory, but they are free to mutate their own memory. In some sense actors are like function in functional languages. In other words they're stateless.

In some ways Actor model is similar to [[Thread pool]].