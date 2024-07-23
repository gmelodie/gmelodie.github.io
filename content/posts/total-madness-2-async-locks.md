+++
public = "false"
date = "2024-07-25"
title = "Total Madness #2: Async Locks"
series = ["Total Madness"]
+++


On the first episode we discussed why Locks are important and some associated issues with them. Last time we unraveled the inners of the Async/Await concurrency model. Now, let us finally merge these two topics to explore Async Locks: a way to overcome some traditional issues with locks using the async/await model. Finally, we'll explore the difference between Rust's `std::sync::Mutex` and `tokio::sync::Mutex` and see how this relates to async locks.

# Episode 2: Async Locks

As a reminder, let's review our original example and see why we need locks at all in the first place:

> Two siblings want to use the toothbrush they share, but obviously only one of them can use it at a time. The childish, sibling-like thing to do is to say “I’m first!” before the other person. The rule is simple: whoever starts saying “I’m first” first, gets to use it.

A lock, or `Mutex`, is just a way to allow tasks to do that. In this analogy, saying "I'm first!" is analogous to `mutex.lock()`, so if a task acquires a mutex, all other tasks trying to acquire it wait until the first one is done (i.e. when it calls `mutex.unlock()`, or `mutex.release()` or whatever name the developers decided to use).


As we saw on Episode 0, it is all fun and games until we face `Deadlocks`, which is when tasks get stuck waiting forever on each other. Several different situations can cause a deadlock, but a very common one is interrupts. Let's see an example of interrupts causing a deadlock from the first episode:
```rust
fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock();
    // gabriel brushes teeth
    // <--- INTERRUPT (Fefê)
    tb_lock.unlock();
}


fn interrupt() {
    if tb_lock.lock() {
        // brush cousins' teeth...
    }
}
```

In the above example, the `main()` function is interrupted right after locking the toothbrush, but the `interrupt()` function needs to lock `tb_lock` as well, which causes the interrupt code to be deadlocked. This is a typical situation of: "your computer stopped working", because interrupts usually can't be themselves interrupted, so since they can't be interrupted and are stuck, there's nothing to help the computer regain control of execution. We talked about a way to avoid this in the first post: instead of using `lock()`, we use `try_lock()`, which returns either `true` if it acquired the lock, or `false` otherwise. Either way, the function returns. It doesn't **block** (this is what we call a **non-blocking** function).

> Aside: why interrutps can't be interrupted
>
> Think about a keyboard buffer: when a key gets pressed on the keyboard, the processor generates an interrupt so that the kernel can get the key from the keyboard's memory, which could very well only hold a maximum of one key) and put it in a buffer (i.e. an array/vector). If the code that copies the key gets interrupted by another key press even, they key is lost.


An async lock is an alternative to the `try_lock()` idea using, you guessed it, `async/await`. Here's more or less what we'll do to the original `lock()` code, can you tell why would this be better than the first approach?
```rust
async fn main() {
    let mut tb_lock = ToothbrushLock::new();
    tb_lock.lock().await;
    // gabriel brushes teeth
    // <--- INTERRUPT (Fefê)
    tb_lock.unlock();
}


async fn interrupt() {
    tb_lock.lock().await;
    // brush cousins' teeth...
}

```

Well the first obvious advantage is that you can guarantee that a lock will be locked. I mean think about it, in this snippet of the `try_lock()` example, I actually hid something very bad, can you figure out what it is?
```rust
fn interrupt() {
    if tb_lock.try_lock() {
        // brush cousins' teeth...
    }
}
```

It's the `else` statement! The big down side of `try_lock`ing is that, if you can't `lock()`, then you just... deal with it. There are no guarantees that `try_lock()` will succeed, and the big advantage of async locks is that you can tell the computer: "Can I lock now? Not yet? That's alright, I'll just let someone else run and try again later". Let's now see how an implementation of async locks would look.


## Implementing an async lock
We're going to get somewhat Rust specific, but the errors raised by the compiler will help us understand issues and limitations of async locks. In this sense, getting a bit Rust specific is a feature, not a bug!


First, let's create some boilerplate code. This is the code we wrote in the last post for our toothbrushing example:
```rust
mod executor;
mod task;

use executor::Executor;
use futures::pending;
use task::Task;

async fn brush_teeth(times: usize) -> String {
    for i in 0..times {
        // TODO: lock
        println!("Brushing teeth {}", i);
        pending!(); // new!
    }
    return "Done".to_string();
}

fn main() {
    let mut executor = Executor::new();

    let task_gabe = Task::new("gabe".to_string(), brush_teeth(20));
    let task_nat = Task::new("nat".to_string(), brush_teeth(30));
    let task_fefe = Task::new("fefe".to_string(), brush_teeth(100));

    executor.tasks.push(task_gabe);
    executor.tasks.push(task_nat);
    executor.tasks.push(task_fefe);

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            print!("{}: ", task.name);
            task.poll();
        }
        println!("--- Went through all tasks once");

        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```

Now, what we want to do is change it to have a `ToothbrushLock`. For this, let's start with the code we wrote in the first episode:

```rust
struct ToothbrushLock {
    locked: bool,
}

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

And finally, adding the two:
```rust
async fn brush_teeth(brush_lock: &mut ToothbrushLock, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock(); // new!
        println!("Brushing teeth {}", i);
        pending!();
        brush_lock.unlock(); // new!
    }
    return "Done".to_string();
}

fn main() {
    let mut executor = Executor::new();

    let mut brush_lock = ToothbrushLock::new(); // new! (literally)

    let task_gabe = Task::new("gabe".to_string(), brush_teeth(&mut brush_lock, 20));
    let task_nat = Task::new("nat".to_string(), brush_teeth(&mut brush_lock, 30));
    let task_fefe = Task::new("fefe".to_string(), brush_teeth(&mut brush_lock, 100));

    executor.tasks.push(task_gabe);
    executor.tasks.push(task_nat);
    executor.tasks.push(task_fefe);

// ... (rest of the code)
```

There's nothing async in our lock for now, not to worry though, we'll get there soon enough. First let's get this code to compile, because as of now it doesn't. Let's see why:
```bash
error[E0597]: `brush_lock` does not live long enough
  --> src/main.rs:27:63
   |
25 |     let mut brush_lock = ToothbrushLock::new(); // new! (literally)
   |         -------------- binding `brush_lock` declared here
26 |
27 |     let task_gabe = Task::new("gabe".to_string(), brush_teeth(&mut brush_lock, 20));
   |                     ------------------------------------------^^^^^^^^^^^^^^^------
   |                     |                                         |
   |                     |                                         borrowed value does not live long enough
   |                     argument requires that `brush_lock` is borrowed for `'static`
...
45 | }
   | - `brush_lock` dropped here while still borrowed
```


This error happens three times, one for each call of `Task::new()`, but I've redacted the other two since they're the same. The problem here is that, since `brush_teeth()` is a `Future`, it doesn't run right away. Actually, it can run whenever! The compiler doesn't know when it'll run, much less when it'll finish. As a result, it can't guarantee that it will finish running before `main()` finishes (which is when `brush_lock` will be freed).

> **Scopes:** scopes are how the Rust compiler knows which variables get discarted/freed/**dropped**. If a variable is declared inside of a function (e.g. `brush_lock` inside of `main()`), it will belongs to the function's scope, and therefore will be dropped when the scope of the function ends (a.k.a.: in line 45).

One way around this is to use **Reference Counting (RC)**, or, even better **Atomic Reference Counting (ARC)**, which is the same thing but with atomic operations. `Arc` allows us an object to have several owners, and, when the scopes of all the owners of the `Arc` end, only then the `Arc` is dropped. Here's how to use it:

```rust
async fn brush_teeth(brush_lock: Arc<ToothbrushLock>, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock(); // new!
        println!("Brushing teeth {}", i);
        pending!();
        brush_lock.unlock(); // new!
    }
    return "Done".to_string();
}

fn main() {
    let mut executor = Executor::new();

    let brush_lock = Arc::new(ToothbrushLock::new()); // new! (literally)

    let task_gabe = Task::new("gabe".to_string(), brush_teeth(brush_lock.clone(), 20));
    let task_nat = Task::new("nat".to_string(), brush_teeth(brush_lock.clone(), 30));
    let task_fefe = Task::new("fefe".to_string(), brush_teeth(brush_lock.clone(), 100));

    executor.tasks.push(task_gabe);
    executor.tasks.push(task_nat);
    executor.tasks.push(task_fefe);
```
As you can see, it's fairly simple to use `Arc`, just enclose our thing with it and it should do its magic! Unfortunately for us, this is not the only issue we have to deal with. Here's what we get when we try to compile now:

```bash
error[E0596]: cannot borrow data in an `Arc` as mutable
  --> src/main.rs:14:9
   |
14 |         brush_lock.lock(); // new!
   |         ^^^^^^^^^^ cannot borrow as mutable
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Arc<ToothbrushLock>`

error[E0596]: cannot borrow data in an `Arc` as mutable
  --> src/main.rs:17:9
   |
17 |         brush_lock.unlock(); // new!
   |         ^^^^^^^^^^ cannot borrow as mutable
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Arc<ToothbrushLock>`

For more information about this error, try `rustc --explain E0596`.
error: could not compile `episode-2-async-locks` (bin "episode-2-async-locks") due to 2 previous errors
```



---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
- [The Little Book of Rust Macros](https://veykril.github.io/tlborm/)

