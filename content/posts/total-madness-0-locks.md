+++
date = "2024-06-17"
title = "Total Madness #0: Locks"
series = ["Total Madness"]
series_order = 1
+++

This is the first post in a series about my urges to figure out the dark magics of the computer world. You see, I recently have had some free time on my hands, and I decided to spend it to scratch some itches I've had for as long as I can code (writing an [OS in Rust](https://github.com/gmelodie/cruzos), for instance). As I dove deeper into the dark magics, I discovered the truth about things I never really liked to assume are true (but did it anyways, for the sake of sanity, at the time).

I don't care anymore about sanity. Let's talk architectures, let's talk memory, let's talk concurrency until we drown in deadlocks, let's find out why there can never be an address 0x0, and yet there's always one. Those who read up beware: this is no place for the sane. Buckle up, for we're going towards Total Madness.

# Episode 0: Locks

Let's start with an easy one: Locks. Gotcha! One of the most important lessons we'll learn here is this: absolutely nothing is as easy as it locks (pun intended). But seriously now, let's reflect on what a lock is. Ignore all the textbook stuff you may or may not know and think about what a lock would look like.

**Obs:** A lock is also called a `Mutex`, short for `Mutual Exclusion`.

Here are my personal initial thoughts, they come as a gross example: two siblings want to use the toothbrush they share, but obviously only one of them can use it at a time. The childish, sibling-like thing to do is to say "I'm first!" before than the other person. The rule is simple: whoever starts saying "I'm first" first, gets to use it. Now, there would be an argument to be made about mili or even nanoseconds of difference (clearly two humans cannot tell which sibling started saying it first if the time difference is mili or nanoseconds appart). Thankfully this is not an issue with typical computers as the sentence would be an instruction running on the computer, so whichever sibling's instruction runs first wins.

Here's a simple implementation of it in Rust (you don't need to know Rust to follow up, you just need to get the general idea of the snippets which should be straightforward for anyone with some general programming experience):

```rust
struct ToothbrushLock {
    locked: bool,
}
```

That's it! We only need a variable (`locked`) that says if the lock is being used, we call this `locked`, (`locked == true`) or not (`locked == false`). Now for the functionality part. In Rust we can write functions that belong to a type, in this case our struct `ToothbrushLock` like so:

```rust
impl ToothbrushLock {
    fn new() -> Self {
        Self { locked: false }
    }
}
```

This function returns an instance of `ToothbrushLock`. We call it in our `main` function like so:

```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();
}
```

We need the `mut` to tell rust that this is a mutable variable, since we'll need to change (mutate) it. In fact, let's do this now. We'll add a function to lock `ToothbrushLock`, and another function to unlock it:

```rust
impl ToothbrushLock {
    fn new() -> Self {
        Self { locked: false }
    }

    fn lock(&mut self) {
        self.locked = true;
    }
    fn unlock(&mut self) {
        self.locked = false;
    }
}
```

Here's how we'd use these new functions:

```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock();
    // brush teeth
    tb_lock.unlock();
}
```

This time the functions take a `&mut self`, which is a mutable reference to an instance of `ToothbrushLock`, and change the `locked` variable to `true` (when `lock`ing) and to `false` (when `unlock`ing). Neat! But there's an issue with this approach. Can you spot it? *Really* try to figure it out on your own before reading on!

**Obs**: the `lock()` and `unlock()` functions can also be called `acquire()` and `release()` respectvely.

Ok, ready?

Here it is: the `lock` function doesn't check if the lock is already locked! This means that we can have this horrific case:

```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();

    tb_lock.lock(); // gabriel locks toothbrush
    // gabriel starts brushing teeth
    tb_lock.lock(); // nathan locks toothbrush
    // nathan starts brushing tee- OH NOOOOOOOOOOOO
    tb_lock.unlock();
}
```

**Obs**: Nathan is my brother, in case you're wondering.

This is no good! Our lock is basically doing nothing.

## Spinlocks

The idea behind the solution to this issue is what most locks do: the `lock()` function **blocks** until the lock is locked, like so:


```rust
impl ToothbrushLock {
    fn new() -> Self {
        Self { locked: false }
    }

    // updated lock function!
    fn lock(&mut self) {
        while self.locked == true {} // keep looping until locked is false
        self.locked = true;
    }


    fn unlock(&mut self) {
        self.locked = false;
    }
}
```

This is called a **Spinlock** because it *spins* in a loop until it gets the lock, and it works really well. However, there are a couple issues with this approach, can you figure out what those are? Again, *reeeeally* try to figure it out on your own before moving on.


Let's check out our usage from before. What happens when Nathan tries to lock `tb_lock` now?

```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();

    tb_lock.lock(); // gabriel locks toothbrush
    // gabriel starts brushing teeth
    tb_lock.lock(); // nathan locks toothbrush
    tb_lock.unlock();
}
```

It'll never make it after the second `tb_lock.lock() // nathan locks toothbrush`, because this `lock()` waits until `locked == false`, which it will never be since it needs `unlock` to be called, and `unlock` doesn't get called until after this `lock`. We've reached the infamous `Deadlock`! A deadlock occurs when a resource tries to `lock()` something that will never unlock. We won't go into much detail regarding deadlocks

Now you must be thinking: "Gabe, you're a moron! You just create another thread for Nathan!", and you'd be right about one of two things there. We didn't talk about threads (yet!), and so let's assume that in our current tooth brushing world there is no such thing as a thread (yet!!). Our computer can only run a single instruction at a time, which begs the question: "Then we don't need locks at all, right?". You see, just because we don't run two things at the same time it doesn't mean that our computer can't *pretend* like it does. Yes sure, our computer doesn't have threads, but you know what it does have? Interrupts!


## Interrupts: a lock's nightmare!

If you know me, you know I absolutely hate interrupts. I mean really, they're so annoying! They think they're so important with their news they think everyone can jus-

Interrupt:
> A KEY HAS BEEN PRESSED DEAL WITH IT RIGHT NOW
>
> Ass. Mr. Rude Interrupt


The above is a good example of an interrupt. An interrupt is the processor's way of letting you know that there's important priority work that needs to be handled, like a key press on a keyboard. Key presses happen very rarely (even if you type 300 words per minute, that's still very slow for the processor), and so you want to deal with them as soon as possible. Almost all I/O operations can be handled with interrupts: disk reads and writes, network packets, etc.

The way I see it, an interrupt is just like another thread, because for the running program that gets interrupted, it's like the entire interrupt happened in an infinitesimal moment in time. But what does all of this mean for our spinlocks? Let's look at an example where my aunt Fefê yanks the toothbrush off my teeth to brush my counsins' teeth. Can you see the problem here?

```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock();
    // gabriel brushes teeth
    // <--- INTERRUPT (Fefê)
    tb_lock.unlock();
}


fn interrupt() {
    tb_lock.lock();
    // brush cousins' teeth...
}
```

Again we've reached a deadlock: the interrupt will never end because it's waiting for the lock to be released. However, the lock will never be released because for that to happen it needs to be unlocked by the code that was interrupted, which it'll never be because... You get the idea. How would you fix this? Again, try to think of a solution before moving on.

One thing we could do is add a function `try_lock()`, which is also very common to locks, to our `ToothbrushLock`, like so:

```rust
impl ToothbrushLock {
    fn new() -> Self {
        Self { locked: false }
    }

    fn lock(&mut self) {
        while self.locked == true {}
        self.locked = true;
    }

    fn try_lock(&mut self) -> bool {
        if self.locked == true {
            return false;
        }
        self.locked = true;
        return true;
    }

    fn unlock(&mut self) {
        self.locked = false;
    }
}
```

The `try_lock()` function returns right away with a boolean value (`true` or `false`) letting the caller know whether it could or could not lock right away. Now we can rewrite our interrupt:

```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock();
    // gabriel brushes teeth
    // <--- INTERRUPT (Fefê)
    tb_lock.unlock();
}


fn interrupt() {
    if tb_lock.try_lock() {
        // brush cousins' teeth...
    }
}
```

Now my aunt will only brush my cousins' teeth if it can lock, and will just return otherwise (perhaps she'll try again 5 minutes from now, or look for another less gross brush). This also solves our problem with my brother from earlier:
```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock();
    // brush teeth
    if tb_lock.try_lock() { // nathan tries to lock toothbrush
         // nathan brushes teeth
    }
    tb_lock.unlock();
}
```

Of course we'd need more code to have my brother and my aunt try again later if they don't get the lock, but the deadlocks are gone, which is great! However there's again another problem we've ignored. This one is a bit harder to spot, but I'll have you try to anyways, can you see it?

Ok here it is: what if...
```rust
impl ToothbrushLock {
    fn new() -> Self {
        Self { locked: false }
    }

    fn lock(&mut self) {
        while self.locked == true {}
        self.locked = true;
    }

    fn try_lock(&mut self) -> bool {
        if self.locked == true {
            return false;
        }
        // <----------------- INTERRUPT
        self.locked = true;
        return true;
    }

    fn unlock(&mut self) {
        self.locked = false;
    }
}
fn main() {
    let mut tb_lock = ToothbrushLock::new();
    if tb_lock.try_lock() { // nathan locks toothbrush
         // nathan brushes teeth
    }
}
```

To imagine this on our tooth brushing analogy, think about aunt Fefê's interruption like a time stopping device. So the sequence of events would be like so:
1. Nathan calls `try_lock()`.
2. Inside `try_lock()`, Nathan checks if `locked == false`, which it is.
3. At this point, after Nathan has checked the locked but right before he locks it, aunt Fefê stops time (interrupt) and calls `try_lock()` herself.
4. Inside `try_lock()`, aunt Fefê checks if `locked == false`, which it is.
5. Aunt Fefê sets `locked` to `true`.
6. The interrupt ends without `unlock`ing.
7. Nathan sets `locked` to `true` thinking it's available (which isn't, it's locked by aunt Fefê).
8. Nathan brushes tee- oh no! Two people brushing their teeth at the same time?


Of course one could say: "Well, then just make sure interrupts release all locks before they return then!", but as now professional lock designers, we can't ship our `ToothbrushLock` with a note saying "This is a great lock! However you have to be REEEEEEALLY careful wehn using it." I mean come n, we can surely better than that.


## Atomics

Do you know what an atom is? It's been a hot topic in Chemistry for a long time, and nowadays we know a lot about them: they're tiny things that make the world around us. But did you know that atoms are not *atomic*? Here's a definition of atomic:

> The smallest part of an element that can exist

We now know that what Chemistry and Physics call atoms are not really atoms, because they can be split apart, etc. But what does that have to do with locks, computers and code? In Systems Design, an *atomic* instruction is an instruction that cannot be split. You see, the issue with our `try_lock()` is that we need to check if `locked == true` and, if not, update it, all of that without being interrupted in the middle. We need this process to be *atomic*. Luckily, most modern computer architectures offer indivisible instructions to do that. On x86 (which is Intel's architecture) for instance, this instruction is called `compareAndSwap` or `compareAndExchange`. Here's the pseudocode for it in Rust:

```rust
fn compare_and_exchange(original: &mut bool, expected: bool, new: bool) -> {
    if *original == expected {
        *original = new;
        return true;
    }
    return false;
}
```
**Obs**: The actual `compareAndExchange` and `compareAndSwap` instructions take integer numbers instead of booleans, but as `bool`s and `int`s are generally equivalent (booleans are usually implemented using `int`s), I've decided to use `bool`s for the sake of simplicity in this example.

This function gets two boolean values `original` and `expected` and, if they're the same, it updates `original` with the value of `new`. Here's how we'd use it:
```rust
fn compare_and_exchange(original: &mut bool, expected: bool, new: bool) {
    if *original == expected {
        *original = new;
    }
}

fn main() {
    let mut original = false;
    compare_and_exchange(&mut original, false, true);
    println!("{original}");                         // prints "true"
}
```

Here's how our `try_lock()` would look with this function:
```rust
fn try_lock(&mut self) -> bool {
    let expected = false;
    let new = true;
    return compare_and_exchange(&mut self.locked, expected, new);
}
```

**Obs**: in rust, we'd implement *actual* atomics using the `Atomic` types. Check out the [Appendix I](#appendix-i-atomics-in-rust) below for an actual implementation.


There's also another interesting approach for locks that allows us to entirely ditch the `try_lock` function, but I believe we've had enough madness for today. The next post will prepare us with the concepts we need to understand these locks. Then we'll move on to implement them on the third post of the series.



## Appendix I: Atomics in rust
Here's how to actually implement our locks in rust, using Atomic types:
```rust
use std::sync::atomic::{AtomicBool, Ordering};

struct ToothbrushLock {
    locked: AtomicBool,
}

impl ToothbrushLock {
    fn new() -> Self {
        Self {
            locked: AtomicBool::new(false),
        }
    }

    fn lock(&mut self) {
        while self.locked.load(Ordering::SeqCst) {}
        self.locked.store(true, Ordering::SeqCst);
    }

    fn try_lock(&mut self) -> bool {
        let expected = false;
        let new = true;
        match self
            .locked
            .compare_exchange(expected, new, Ordering::SeqCst, Ordering::SeqCst)
        {
            Ok(_) => return true,
            Err(_) => return false,
        }
    }

    fn unlock(&mut self) {
        self.locked.store(false, Ordering::SeqCst);
    }
}

fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock();
    // brush teeth
    if tb_lock.try_lock() { // nathan locks toothbrush
         // nathan brushes teeth
    }
    tb_lock.unlock();
}
```




---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
- [Writing an OS in Rust](https://os.phil-opp.com/async-await/)
- [Atom definition (Oxford Reference)](https://www.oxfordreference.com/display/10.1093/oi/authority.20110803095432229)
