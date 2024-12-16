+++
title = "LLQ: A wait-free SPSC linked-list queue with recyclable nodes"
date = "2022-12-04T15:39:00-06:00"
+++

Last year, I published a Rust library called [basedrop](/posts/basedrop/), which implements a memory reclamation system tailored to the constraints of real-time audio scenarios. The purpose of basedrop is to make it easy to share dynamically allocated memory with a real-time audio thread while ensuring that no allocations or deallocations happen on that thread. This is accomplished by providing a set of smart pointers (analogous to `Box` and `Arc` from the Rust standard library) which do not directly free their associated allocation when dropped, but instead automatically push it onto a lock-free queue to be collected later on another thread.

Basedrop's design has some compelling benefits: it frees you from having to write code by hand every time you want to transfer an object to another thread to be freed, and if you restrict yourself to its vocabulary of smart pointers, it eliminates the possibility of accidentally dropping an allocation on the real-time thread (a mistake which can easily remain invisible if you don't have something like [`assert_no_alloc`](https://github.com/Windfisch/rust-assert-no-alloc) to catch it). However, after talking with some developers trying to make use of basedrop in real projects, it became clear to me that these benefits come at the cost of a somewhat opinionated API, making it difficult to integrate with certain program architectures. I decided that a stripped-down version of the core linked-list queue would probably have some value, and the end result of that was the [llq](https://github.com/micahrj/llq) crate.

<!--excerpt-->

A central piece of basedrop's design is the [`Node<T>`](https://docs.rs/basedrop/0.1.2/basedrop/struct.Node.html) type, which represents a node that can potentially be added to the collector queue's linked list. Each of basedrop's smart pointers allocates a `Node<T>` on the heap at creation time, so when the time comes to mark the contained object as ready for deallocation, that node already exists, stored inline as part of the original allocation, and can simply be linked into the queue. This makes it possible to send an object back from the real-time thread to be reclaimed without performing any allocator operations.

The llq crate extracts just that core functionality, of a wait-free linked-list queue with preallocated nodes, and presents it in an unopinionated way. With llq, you can create some nodes:

```rust
use llq::{Node, Queue};

let x = Node::new(0);
let y = Node::new(1);
let z = Node::new(2);
```

push them onto a queue:

```rust
let (mut tx, mut rx) = Queue::<usize>::new().split();

tx.push(x);
tx.push(y);
tx.push(z);
```

pull them off the other end:

```rust
let x = rx.pop().unwrap();
let y = rx.pop().unwrap();
let z = rx.pop().unwrap();
```

and even reuse them with a separate queue:

```rust
let (mut tx2, mut rx2) = Queue::<usize>::new().split();

tx2.push(x);
tx2.push(y);
tx2.push(z);
```

and none of the above `push` or `pop` operations will ever allocate or free memory, lock a mutex, or even enter an unbounded compare-exchange loop.

It's worth noting that essentially the only synchronization operations in the entire source of llq are a single [acquire load](https://github.com/micahrj/llq/blob/f5707bd832144308b3482c56b088b3076ea3dd25/src/lib.rs#L193) in the body of `pop` and a [release store](https://github.com/micahrj/llq/blob/f5707bd832144308b3482c56b088b3076ea3dd25/src/lib.rs#L228) in the body of `push`. I consider it a pretty compelling demonstration of Rust's type system and safety guarantees that a concurrent data structure with such minimal synchronization overhead can still have a [data race](https://doc.rust-lang.org/nomicon/races.html)-free public API (assuming llq's implementation is bug-free, of course!).

For reference, the queue design in llq is based on a particular [unbounded SPSC queue design](https://www.1024cores.net/home/lock-free-algorithms/queues/unbounded-spsc-queue) from 1024cores. There's also an implementation of a [similar queue design](
https://github.com/rust-lang/rust/blob/481971978fda83aa7cf1f1f3c80cfad822377cf2/library/std/src/sync/mpsc/spsc_queue.rs) in the internals of the Rust standard library.

It's important to note that llq is designed for a specific, uncommon set of requirements, and several aspects of its design are more or less the opposite of what one would want out of a general-purpose channel for communicating between threads. llq is designed for the scenario where

- you don't want the channel to interact with the OS scheduler at all (no blocking)
- you want to ensure that one thread never allocates or deallocates in the process of sending or receiving items
- you want sending to be infallible for one thread (for e.g. returning objects to be deallocated from the audio thread)

A general-purpose queue should probably

- have blocking semantics and interact with the system scheduler, when receiving from an empty channel or sending to a full channel
- store items inline together, for less pointer-chasing and less memory usage overall
- have a bounded capacity, for backpressure

So, if you're just looking for a generic SPSC queue, there's good chance llq is not what you want. But if you're implementing real-time audio software, or are otherwise facing a situation where some threads in your program have much stricter latency requirements than others, llq might be worth a look.

You can check it out over on [GitHub](https://github.com/micahrj/llq) or on [crates.io](https://crates.io/crates/llq).

Discuss on: [Twitter](https://twitter.com/glowcoil/status/1599518887871008768) · [Mastodon](https://post.lurk.org/@glowcoil/109457572953430604) · [r/rust](https://www.reddit.com/r/rust/comments/zcm465/llq_a_waitfree_spsc_linkedlist_queue_with/?) · [r/programming](https://www.reddit.com/r/programming/comments/zcmcrp/llq_a_waitfree_spsc_linkedlist_queue_with/) · [HN](https://news.ycombinator.com/item?id=33858117)
