+++
title = "Basedrop: A garbage collector for real-time audio in Rust"
date = "2021-04-26T10:18:00-05:00"
+++

In real-time audio, deadlines are critical. Your code has on the order of several milliseconds to fill a buffer with samples to be shipped off to the [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter), milliseconds which it may be sharing with a number of other audio plugins. If your code takes too long to produce those samples, there are no second chances; they simply won't get played, and the user will hear an objectionable glitch or stutter instead.

In order to prevent this, real-time audio code must avoid performing any operations that can block the audio thread for an unbounded or unpredictable amount of time. Such operations include file and network I/O, memory allocation and deallocation, and the use of locks to synchronize with non-audio threads; these operations are not considered "real-time safe." Instead, operations like I/O and memory allocation should be performed on other threads, and synchronization should be performed using primitives that are wait-free for the audio thread. A more thorough overview of the subject can be found in Ross Bencina's now-classic blog post ["Time Waits for Nothing"](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing).

Given that audio software generally does need to allocate memory and make use of it from the audio thread, the question becomes how to accomplish this in a manageable and efficient way while subject to the above constraints. [Basedrop](https://github.com/micahrj/basedrop) is my attempt at providing one answer to this question.

<!--excerpt-->

# Deferred reclamation

Consider a simple scenario: we have a buffer of samples stored in a `Vec<f32>`, possibly synthesized or loaded from disk, and we would like to use it from the audio thread. As an initial sketch of a solution, we could use a wait-free bounded-capacity [SPSC](http://www.1024cores.net/home/lock-free-algorithms/queues) channel (such as the [`rtrb` crate](https://crates.io/crates/rtrb)) to send the buffer over to the audio thread, and then when we're done using it and want to reclaim the memory, we could send it back to a non-real-time thread over another SPSC channel to be freed.

In simple cases, this solution works well. However, it has drawbacks as an application grows in complexity. For instance, if a large number of allocations are being transferred to and from the audio thread, the fixed-capacity channel for returning allocations can fill up. Since it is not acceptable to block the audio thread in this case, the application needs to ensure either that the channel is polled frequently enough to keep up, that the channel always has worst-case capacity (using a more complex dynamically allocated design), or that the audio thread can continue without error if it is not currently possible to send back an allocation. Additionally, this solution relies on programmer discipline to ensure that allocations are always sent back to be freed, and Rust's [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) design makes mistakes in this regard largely invisible. Diagnostic tools like the [`assert_no_alloc` crate](https://crates.io/crates/assert_no_alloc) can go a long way towards detecting such mistakes, but it would be nice to have a guarantee at compile time.

Basedrop's solution is to replace the fixed-capacity ring buffer for returning allocations with an [MPSC](http://www.1024cores.net/home/lock-free-algorithms/queues) linked-list queue whose nodes are created at allocation time for (and stored inline next to) any piece of memory intended to be shared with the audio thread. When the audio thread is ready to release a piece of memory for reclamation, the corresponding node can be pushed onto the queue in an allocation-free, wait-free operation. This pattern is encapsulated by a pair of smart pointers, `Owned<T>` and `Shared<T>`, analogous to `Box<T>` and `Arc<T>`, which push their contents onto the queue for deferred reclamation rather than dropping them directly. The queue can then be processed periodically on another thread using basedrop's `Collector` type.

This system has the advantage that is impossible for the reclamation channel to become full (short of a full-on [OOM](https://en.wikipedia.org/wiki/Out_of_memory)). It is also impossible to forget to send something back to be collected, as long as it was initially wrapped in an `Owned<T>` or `Shared<T>`. `Shared<T>` in particular opens up exciting possibilities for sharing immutable and persistent data structures between audio and non-audio threads in ways that would be cumbersome or impossible with the manual message-passing approach.

# `SharedCell`

Basedrop provides another primitive for sharing memory with the audio thread, called `SharedCell<T>`. `SharedCell<T>` acts as a thread-safe mutable memory location for storing `Shared<T>` pointers, providing `get`, `set`, and `replace` methods (much like `Cell`) for fetching and updating the contents. I envision this being used as a way for a non-real-time thread to atomically publish data which can then be immutably observed by the real-time audio thread.

The main difficulty in implementing this pattern in a lock-free way lies in the fact that getting a copy of a reference-counted pointer actually consists of two steps: first, fetching the actual pointer, and then incrementing the reference count. In between these two steps, writers must not be allowed to replace the pointer with a new value, decrement the reference count for the previous value to zero, and then free its referent, as this would result in a use-after-free for the reader. There are various possible solutions to this problem with different tradeoffs.

The approach taken by `SharedCell<T>` is to keep a reader count alongside the stored pointer. Readers increment this count while fetching the pointer and only decrement it after successfully incrementing the pointer's reference count. Writers, in turn, after replacing the stored pointer, spin until the count is observed to be zero before they are allowed to move on and possibly decrement the reference count. This scheme is designed to be low-cost and non-blocking for readers, while being somewhat higher-overhead for writers, which I deem to be the appropriate tradeoff for real-time audio, where the reader (the audio thread) has much tighter latency deadlines and executes much more often than the writer.

# Future work

Basedrop doesn't currently support dynamically sized types, like `Owned<[T]>` or `Owned<dyn Trait>`. This should become possible when [`CoerceUnsized`](https://doc.rust-lang.org/nightly/core/ops/trait.CoerceUnsized.html) or equivalent is stabilized. For now, it can be worked around without much issue by wrapping the DST in another layer of allocation.

Additionally, `Shared<T>` doesn't currently support weak references for cyclic data structures the way `Arc<T>` does. This would complicate the reference-counting logic (see the [`Arc` source](https://github.com/rust-lang/rust/blob/5702cfa2551a56172a4e392aab4b494562242f35/library/alloc/src/sync.rs)), and I wanted to start with something simple that I could be sure was correct. However, this would certainly be nice to have.

I would also like to explore memory reclamation strategies with less overhead than reference counting, such as the [RCU](https://www.kernel.org/doc/html/latest/RCU/whatisRCU.html) pattern found in the Linux kernel, [epoch-based reclamation](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-579.html), and [quiescent state-based reclamation](https://preshing.com/20160726/using-quiescent-states-to-reclaim-memory/). I haven't yet been able to come up with a design in this vein that both dovetails with Rust ownership and satisfies the constraints of real-time audio (and audio plugins), but I think it's a promising direction for the future.

# Finally

Basedrop is available on [crates.io](https://crates.io/crates/basedrop)! Please feel free to give it a try in your own projects. Feedback and bug reports are welcome.

Lastly, I would like to thank William Light for some very helpful conversations while I was working out the design of basedrop.
