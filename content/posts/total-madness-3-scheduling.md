+++
date = "2024-06-18"
title = "Total Madness #3: Scheduling"
series = ["Total Madness"]
series_order = 4
+++

# Episode 3: Scheduling
Welcome back! I have to say if you went through all of the last post and *still* want to know more about what I have to say, you might just be as masochist as I am. Well then, let's get on with our madness dose for the day. Today we're talking about Scheduling.


## Cooperative Multitasking

Last time we saw how to divide tasks so that we could run a little piece each task and keep swapping between them. That's called "cooperative multitasking". "Multitasking" because we're jiggling tasks back and forth, essentially doing a bunch of things at the same time, and "cooperative" because it's up to our tasks to yield back (either with `Poll::Ready` or with `Poll::Pending`) to our `Executor`. You may have noticed how the "cooperative" part can be an issue: tasks may never yield back and steal all our processing forever. But that's not all. You may be surprised to learn that the "multitasking" part can *also* be problematic. How so? Think about it.

problem with multitasking: a task may run for a long time before yielding

## Scheduling
## Ticket Scheduling





---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
- [Writing an OS in Rust](https://os.phil-opp.com/async-await/)
- [Atom definition (Oxford Reference)](https://www.oxfordreference.com/display/10.1093/oi/authority.20110803095432229)
