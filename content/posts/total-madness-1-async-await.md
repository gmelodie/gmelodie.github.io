+++
date = "2024-06-08"
title = "Total Madness #1: Async/Await"
series = ["Total Madness"]
draft = true
+++

blabla intro
# Episode 1: Async/Await

## Async locks
As of now, we have a decent enough `ToothbrushLock` implementation, but one thing still annoys me: there is no way besides `try_lock()` to ensure that there won't be a deadlock. I mean think about it, if we call lock and someone else has the lock, and we are in a single-threaded computer/processor, there is no way we can recover from that deadlock (besides of course rebooting our machine). One way to overcome this would be to use the async/await model.


## Ticket locks
## Scheduling
## Ticket Scheduling


blabla
```rust
struct ToothbrushLock {
    locked: bool,
}
```

---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
