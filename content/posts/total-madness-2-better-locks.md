+++
date = "2024-06-18"
title = "Total Madness #2: Better Locks"
series = ["Total Madness"]
series_order = 3
+++

# Episode 2: Better Locks

## Recap Locks
## Recap Async

## Async locks
As of now, we have a decent enough `ToothbrushLock` implementation, but one thing still annoys me: there is no way besides `try_lock()` to ensure that there won't be a deadlock. I mean think about it, if we call lock and someone else has the lock, and we are in a single-threaded computer/processor, there is no way we can recover from that deadlock (besides of course rebooting our machine). One way to overcome this would be to use the async/await model.



---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
- [Writing an OS in Rust](https://os.phil-opp.com/async-await/)
- [Atom definition (Oxford Reference)](https://www.oxfordreference.com/display/10.1093/oi/authority.20110803095432229)
