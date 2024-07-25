+++
public = "false"
date = "2024-07-25"
title = "Total Madness #2: Async Locks"
series = ["Total Madness"]
+++


In the [first episode](./total-madness-0-locks.md) we discussed why Locks are important and some associated issues with them. [Last time](./total-madness-1-async-await.md) we unraveled the inners of the Async/Await concurrency model. Now, let us finally merge these two topics to explore Async Locks: a way to overcome some traditional issues with locks using the async/await model. Finally, we'll explore the difference between Rust's `std::sync::Mutex` and `tokio::sync::Mutex` and how this relates to async locks.


**Obs:** I'll use the terms "Lock" and "Mutex" interchangeably (`lock == mutex`), as well as the verbs "Acquire" and "Lock" (`lock == acquire`).

# Episode 2: Async Locks

As a reminder, let's review our original example and see why we need locks at all in the first place:

> Two siblings want to use the toothbrush they share, but obviously only one of them can use it at a time. The childish, sibling-like thing to do is to say “I’m first!” before the other person. The rule is simple: whoever starts saying “I’m first” first, gets to use it.

A lock, or `Mutex`, is just a way to allow tasks to do that. In this analogy, saying "I'm first!" is analogous to `mutex.lock()`, so if a task acquires a mutex, all other tasks trying to acquire it wait until the first one is done (i.e. when it calls `mutex.unlock()`, or `mutex.release()` or whatever name the developers decided to use).


As we saw on [Episode #0: Locks](./total-madness-0-locks.md), it is all fun and games until we face `Deadlocks`, which is when tasks get stuck waiting forever on each other. Several different situations can cause a deadlock, but a very common one is interrupts. Let's see an example of interrupts causing a deadlock from that episode:
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

In the above example, the `main()` function is interrupted right after locking the toothbrush, but the `interrupt()` function needs to lock `tb_lock` as well, which causes the interrupt code to be deadlocked. This is a typical situation of: "your computer stopped working" because interrupts usually can't be themselves interrupted, so since they can't be interrupted and are stuck, there's nothing to help the computer regain control of execution. We talked about a way to avoid this in the first post: instead of using `lock()`, we use `try_lock()`, which returns either `true` if it acquired the lock, or `false` otherwise. Either way, the function returns. It doesn't **block** (this is what we call a **non-blocking** function).

> Aside: why interrupts can't be interrupted
>
> Think about a keyboard buffer: when a key gets pressed on the keyboard, the processor generates an interrupt so that the kernel can get the key from the keyboard's memory, which could very well only hold a maximum of one key) and put it in a buffer (i.e. an array/vector). If the code that copies the key gets interrupted by another key press event, the key is lost.


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

Well, the first obvious advantage is that you can guarantee that a lock will be locked. I mean think about it, in this snippet of the `try_lock()` example, I actually hid something very bad, can you figure out what it is?
```rust
fn interrupt() {
    if tb_lock.try_lock() {
        // brush cousins' teeth...
    }
}
```

It's the `else` statement! The big downside of `try_lock`ing is that, if you can't `lock()`, then you just... deal with it. There are no guarantees that `try_lock()` will succeed, and the big advantage of async locks is that you can tell the computer: "Can I lock now? Not yet? That's alright, I'll just let someone else run, and try again later". Let's now see how an implementation of async locks would look.


## Implementing an async lock
We're going to get somewhat Rust-specific, but the errors raised by the compiler will help us understand the issues and limitations of async locks. In this sense, getting a bit Rust-specific is a feature, not a bug!


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

Now, what we want to do is change it to have a `ToothbrushLock`. For this, let's start with the code we wrote in the [first episode](./total-madness-0-locks.md):

```rust
struct ToothbrushLock {
    locked: bool,
}

impl ToothbrushLock {
    pub fn new() -> Self {
        Self { locked: false }
    }

    pub fn lock(&mut self) {
        while self.locked == true {}
        self.locked = true;
    }

    pub fn try_lock(&mut self) -> bool {
        if self.locked == true {
            return false;
        }
        self.locked = true;
        return true;
    }

    pub fn unlock(&mut self) {
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

There's nothing async in our lock for now, not to worry though, we'll get there soon enough. First, let's get this code to compile because as of now it doesn't. Let's see why:
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

> **Scopes:** Scopes are how the Rust compiler knows which variables get discarded/freed/**dropped**. If a variable is declared inside of a function (e.g. `brush_lock` inside of `main()`), it will belong to the function's scope, and therefore will be dropped when the scope of the function ends (a.k.a.: in line 45).

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

The problem now is that, while `Arc` allows us to have multiple owners to a type, it doesn't allow any of them to *mutate* the data. This is one of the mechanisms that Rust has to ensure memory **safety**: having two or more tasks altering the same memory region can cause data races for the region. As a reminder, take a look at this example, where both `taskA` and `taskB` want to read `var1: bool` and, if `true`, update it to `false`:

1. `taskA` reads `var1`: value is `true`
2. `taskB` reads `var1`: value is `true`
3. `taskA` writes to `var1`: value is `false`
4. `taskB` writes to `var1`: value is `false`

The problem is that one of them should not be updating the variable, since one should run first. Allowing only one mutable reference to the variable solves this issue, but also restricts the code quite a bit. Another solution that allows us to have multiple mutable references is precisely what we've been doing: locks! So now we need a way to ignore this rule, and for that, we'll use `UnsafeCell`s:

```rust
use std::cell::UnsafeCell; // new!

pub struct ToothbrushLock {
    locked: UnsafeCell<bool>, // new!
}

impl ToothbrushLock {
    pub fn new() -> Self {
        Self {
            locked: UnsafeCell::new(false),
        }
    }

    pub fn lock(&self) {
        unsafe {  // new! unsafe
            while *self.locked.get() == true {}
            *self.locked.get() = true;
        }
    }

    pub fn unlock(&self) {
        unsafe { // new! unsafe
            *self.locked.get() = false;
        }
    }
}
```

**Obs:** I've removed the `try_lock()` method since our async lock is trying to get rid of it anyway.

`UnsafeCell`s allow us to have multiple mutable references to a variable, but, as the name suggests, it is `unsafe`, meaning that the borrow checker doesn't apply its memory safety rules to it. In fact, we have to enclose all calls to `self.locked.get()` in `unsafe {..}` blocks. `unsafe {...}` blocks basically tell the compiler to ignore checking for memory safety on these regions, it's our way of saying "don't worry, I know what I'm doing".


And with this change, our code compiles!!! Hooray! But wait, what is this output?
```
gabe: Brushing teeth 0
```
The code hangs after printing this first line, try to figure out what is wrong with the code. Here's the solution:
```rust
async fn brush_teeth(brush_lock: Arc<ToothbrushLock>, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock();
        println!("Brushing teeth {}", i);
        brush_lock.unlock();
        pending!();
    }
    return "Done".to_string();
}

// rest of code...
```

The change is minimal but effective: putting the `unlock()` call before `pending!()`. Of course, we need to release the mutex before we return execution flow to the scheduler/executor because if we don't, the next task (in this case `nat` or `fefe`) will be stuck trying to `lock()`. But what we wanted, in the beginning, was precisely a way to return execution flow to the scheduler until we're able to `lock()`. In other words, this is what we want to be able to do:

```rust
async fn brush_teeth(brush_lock: Arc<ToothbrushLock>, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock().await; // try to lock the mutex, continue running other tasks otherwise
        println!("Brushing teeth {}", i);
        // no need for unlock(), mutex is unlocked magically
    }
    return "Done".to_string();
}

// rest of code...
```

We're close, but not quite there yet. As a next step, let's keep the `unlock()` method, and focus on making `lock()` async so we can call `await` on it:
```rust
use futures::pending;
use std::cell::UnsafeCell;

pub struct ToothbrushLock {
    locked: UnsafeCell<bool>,
}

impl ToothbrushLock {
    pub fn new() -> Self {
        Self {
            locked: UnsafeCell::new(false),
        }
    }

    pub async fn lock(&self) { // new! async fn
        unsafe {
            while *self.locked.get() == true {
                pending!(); // new!
            }
            *self.locked.get() = true;
        }
    }

    pub fn unlock(&self) {
        unsafe {
            *self.locked.get() = false;
        }
    }
}
```

With those two small adjustments, we can call the lock like so:
```rust
async fn brush_teeth(brush_lock: Arc<ToothbrushLock>, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock().await;
        println!("Brushing teeth {}", i);
        brush_lock.unlock();
    }
    return "Done".to_string();
}

// rest of code...
```

And our new output is:
```
gabe: Brushing teeth 0
Brushing teeth 1
...
Brushing teeth 18
Brushing teeth 19
nat: Brushing teeth 0
Brushing teeth 1
Brushing teeth 2
Brushing teeth 3
...
Brushing teeth 28
Brushing teeth 29
fefe: Brushing teeth 0
Brushing teeth 1
Brushing teeth 2
Brushing teeth 3
Brushing teeth 4
...
Brushing teeth 98
Brushing teeth 99
--- Went through all tasks once
```

As you can see, we're not circling through tasks. This happens because we only yield execution on the `lock()` method. This means that a task that owns the lock will do:

1. print `Brushing teeth 1`
2. unlock the lock
3. lock/acquire the lock
4. yield execution

Then all the tasks will try to lock, but not have any success and yield execution back. When the scheduler finally goes back to the task that owns the lock, it'll repeat the sequence above. To fix that we can make `unlock()` also yield execution back:

```rust
use futures::pending;
use std::cell::UnsafeCell;

pub struct ToothbrushLock {
    locked: UnsafeCell<bool>,
}

impl ToothbrushLock {
    pub fn new() -> Self {
        Self {
            locked: UnsafeCell::new(false),
        }
    }

    pub async fn lock(&self) {
        unsafe {
            while *self.locked.get() == true {
                pending!();
            }
            *self.locked.get() = true;
        }
    }

    pub async fn unlock(&self) {
        unsafe {
            *self.locked.get() = false;
            pending!(); // new!
        }
    }
}
```

And our call becomes:

```rust
async fn brush_teeth(brush_lock: Arc<ToothbrushLock>, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock().await;
        println!("Brushing teeth {}", i);
        brush_lock.unlock().await;
    }
    return "Done".to_string();
}

// rest of code...
```

Yes! Now our execution is back to being:
```
gabe: Brushing teeth 0
nat: Brushing teeth 0
fefe: Brushing teeth 0
--- Went through all tasks once
gabe: Brushing teeth 1
nat: Brushing teeth 1
fefe: Brushing teeth 1
--- Went through all tasks once
gabe: Brushing teeth 2
nat: Brushing teeth 2
fefe: Brushing teeth 2
--- Went through all tasks once
gabe: Brushing teeth 3
nat: Brushing teeth 3
fefe: Brushing teeth 3
--- Went through all tasks once
gabe: Brushing teeth 4
nat: Brushing teeth 4
fefe: Brushing teeth 4
--- Went through all tasks once
gabe: Brushing teeth 5
nat: Brushing teeth 5
fefe: Brushing teeth 5
...
```

Yes! Now our async lock works. The only last detail we need to take care of is the fact that we need to call `unlock()`. You see, we can use the magic of Rust to make the compiler call `unlock()` for us whenever is appropriate to release the lock (i.e. when the lock goes out of scope). In our example, the scope of the lock is one iteration of the `for` loop, so the unlock would happen exactly where we put it, just before the end of a loop. To do that, we need two things:
1. Have a `ToothbrushGuard` type that is returned by `lock`, which will be in the `for` scope.
2. Implement the `Drop` trait for `ToothbrushGuard`, `Drop` tells the compiler what to do when an object goes out of scope. In our case, we want to make it unlock when `ToothbrushGuard` goes out of scope.
```rust
pub struct ToothbrushGuard<'guard_life> {
    mutex: &'guard_life ToothbrushLock,
}
impl<'guard_life> ToothbrushGuard<'guard_life> {
    pub fn new(mutex: &'guard_life ToothbrushLock) -> Self {
        ToothbrushGuard { mutex }
    }
}
impl Drop for ToothbrushGuard<'_> {
    fn drop(&mut self) {
        self.mutex.unlock();
    }
}
```

And have `lock()` return a guard:
```rust
impl ToothbrushLock {
    pub fn new() -> Self {
        Self {
            locked: UnsafeCell::new(false),
        }
    }

    pub async fn lock(&self) -> ToothbrushGuard {
        unsafe {
            while *self.locked.get() == true {
                pending!();
            }
            *self.locked.get() = true;
            ToothbrushGuard::new(self) // new!
        }
    }

    fn unlock(&self) {
        unsafe {
            *self.locked.get() = false;
            // no longer able to call pending!() here
        }
    }
}
```

With these things in place, we can finally call our functions the way we wanted:
```rust
async fn brush_teeth(brush_lock: Arc<ToothbrushLock>, times: usize) -> String {
    for i in 0..times {
        brush_lock.lock().await;
        println!("Brushing teeth {}", i);
    }
    return "Done".to_string();
}
```

Amazing! There's only one last thing to do though. You see, because the `drop()` function cannot be `async`, and `unlock()` is called in `drop()`, this means that `unlock()` can no longer be `async` (aka it can no longer call `pending!()` to yield execution back. As a result, the previous example does compile and work, but we're not circling through the tasks once again. A quick hack fixes that (try to do that yourself without peeking first!):

```rust
impl ToothbrushLock {
    pub fn new() -> Self {
        Self {
            locked: UnsafeCell::new(false),
        }
    }

    pub async fn lock(&self) -> ToothbrushGuard {
        unsafe {
            pending!(); // return pending before!
            while *self.locked.get() == true {
                pending!();
            }
            *self.locked.get() = true;
            ToothbrushGuard::new(self)
        }
    }

    fn unlock(&self) {
        unsafe {
            *self.locked.get() = false;
        }
    }
}
```
Returning `pending!()` before the `while{}` loop in `lock()` ensures that execution will be yielded back even if the lock was previously acquired by the task.


That's it! Async locks.



## Appendix I: Atomic Async Locks
What we did was all fun and games, but if an interrupt occurs in the middle of the code we wrote, it all goes to shit. Solving this requires atomic operations that guarantee that they won't be interrupted. We did the same thing in [Episode #0: Locks](./total-madness-0-locks.md):
```rust
use futures::pending;
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicBool, Ordering};

pub struct ToothbrushLock {
    locked: UnsafeCell<AtomicBool>,
}

impl ToothbrushLock {
    pub fn new() -> Self {
        Self {
            locked: UnsafeCell::new(AtomicBool::new(false)),
        }
    }

    pub async fn lock(&self) -> ToothbrushGuard {
        unsafe {
            pending!();
            while (&*self.locked.get()).load(Ordering::SeqCst) == true {
                pending!();
            }
            (&*self.locked.get()).store(true, Ordering::SeqCst);
            ToothbrushGuard::new(self)
        }
    }

    fn unlock(&self) {
        unsafe {
            (&*self.locked.get()).store(false, Ordering::SeqCst);
        }
    }
}

pub struct ToothbrushGuard<'guard_life> {
    mutex: &'guard_life ToothbrushLock,
}
impl<'guard_life> ToothbrushGuard<'guard_life> {
    pub fn new(mutex: &'guard_life ToothbrushLock) -> Self {
        ToothbrushGuard { mutex }
    }
}
impl Drop for ToothbrushGuard<'_> {
    fn drop(&mut self) {
        self.mutex.unlock();
    }
}
```


---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
- [Understanding and handling Rust mutex poisoning](https://blog.logrocket.com/understanding-handling-rust-mutex-poisoning/#what-mutex-poisoning)
- [How are mutexes implemented in Rust?](https://whenderson.dev/blog/rust-mutexes/)

