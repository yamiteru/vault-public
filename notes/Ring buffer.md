implementation of a first-in-first-out (FIFO) queue, implemented using a pair of indices into an array of data in memory. Crucially, for efficiency, when data is inserted into the queue, it won’t ever move to another spot in memory. Instead, we move the indices around as data gets added to or removed from the queue.

The array is treated as if one end is connected to the other, creating a ring of data. This means that if these indices are incremented past the end of the array, they’ll go back to the beginning.

To be used in threaded operation it should implement a [[Mutex]].