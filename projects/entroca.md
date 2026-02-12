I'm working on a caching system.

It has a shared-nothing architecture meaning each thread manages its own memory.

There's a router in front of the threads that routes traffic to the right threads.

The caching system will be used for caching database queries, frequently accessed files, etc. (think about where redis/memcached is typically used + CDNs).

The allocated items and their size are unknown ahead of time so the allocator shouldn't be optimized only for X pre-defined sizes.

The allocator will get a pre-allocated memory on startup.

Items in memory need to be allocated and deallocated individually (not in batches or eras).

The allocator should have low fragmentation and should be as fast as possible (using as few CPU instructions as possible).

The allocator will be built from the ground up since I don't want any dependencies so an easy to implement allocator is a nice-to-have.

The allocator don't have to be a real-time allocator but having huge variance in response time is bad so ideally it should be bounded.

The caching system will run on modern hardware. It will be completely in-memory so no spilling to disk.

Show me a table of allocator algorithms (not libraries) I could use.

For each show me big O (both CPU and memory) and estimated number of CPU instructions for alloc and free + estimated LoC for implementation.
