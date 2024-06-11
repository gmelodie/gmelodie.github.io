+++
date = "2024-06-11"
title = "Total Madness #1: Async/Await"
series = ["Total Madness"]
+++

On the last episode we explored a bit of the world of concurrency talking about locks. I mean, we didn't use that word expicitly, but effectively that's what we were talking about. Today we'll explore a crucial concurrency model: async/await.

Oh and don't think I forgot about the promise I made about async locks. We'll talk about that all right, but to do that we first need to understand why async/await is even a thing, and how it works.

# Episode 1: Async/Await

On the last episode I used a gross example to illustrate what happens on a computer when different *tasks* are competing for resources. The word *task* here is an important one. We're talking about theoretical concepts, not what one programming language calls `Task`. For us now, a task is simply a piece of running instructions. Because modern computers need to do a lot of things at once but have limited processors (on our theoretical computer, we only have one processor!), we use tasks to give the users the *impression* that everything is happening all at once. Behind the scenes, the processor jiggles tasks like crazy to create this illusion. But how does it do that? Really think about it, try to figure it out, then continue reading.

## Tasks
Ready? The idea is farily straightforward: we use a `queue` of tasks being run. We cycle through this list and run a part of each task for a little bit of time. Then, when we reach the end of the queue, we go back to the begining and do it all over again.

Now you might be thinking this is it. But in fact a ton of issues can stem from this approach, as well as a ton of ways to solve them. Let's implement a simple task runner following this idea. If you're new to the posts, the examples are in Rust, but you don't need to know Rust at all to follow along, the code is very self-explanatory and I'll explain it afterwards so that anyone with some programming experience can understand them. And plus you might even pick up a new language along the way!

So I'm thinking that we'll eventually need to have a struct holding important information of our executor, but I have no idea what that is at this point, so I'll make an empty struct for now and focus on the methods I want it to have.

```rust
struct Executor {}

impl Executor {
    fn new() -> Self {
        Self {}
    }
}

fn main() {
    let mut executor = Executor::new();
}
```
Okay, we have a struct, a `new()` function/method that returns a new instance of the struct, and a way to call it. The Rust compiler complains of course: "You're not doing anything!", it says. The compiler is right, so let's do something. But, before we do anything, let's also create a `Task` type that our executor will hold in a queue:

```rust
// new!
struct Task {}

// new!
impl Task {
    fn new() -> Self {
        Self {}
    }
}

struct Executor {
    tasks: Vec<Task>, // new: task queue
}

impl Executor {
    fn new() -> Self {
        Self { tasks: Vec::new() } // create new empty task queue
    }
}

fn main() {
    let mut executor = Executor::new();
}
```

I added a `Task` struct and a field on our executor to hold a list of tasks (in a `Vec`). But still, we're not really doing anything with it. Our task should also be able to `run()`, so let's add a method for this:
```rust
impl Task {
    fn new() -> Self {
        Self {}
    }
    fn run(&mut self) {} // new!
}


// ... rest of code
```
Now instead of having only the `new()` method, we also have `run()`, which takes itself as a mutable reference (because it will most likely need to change the `Task` struct it belongs to), but does nothing for now. To run our tasks, we'll add a `for` loop that goes from start to finish of the list, calls `run()` on each task, and do that for as long as we have tasks to run:
```rust
// ... rest of code

fn main() {
    let mut executor = Executor::new();

    // new!
    while executor.tasks.len() != 0 { // run until queue is empty
        for task in executor.tasks.iter_mut() { // for each task in queue...
            task.run(); // run task
        }
    }
}
```

Here we used `iter_mut` to get an [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html) of mutable references (remember our `run()` method needs to be able to mutate the struct. Now we just need to create some tasks to add to our executor:
```rust
// ... rest of code

fn main() {
    let mut executor = Executor::new();

    executor.tasks.push(Task::new()); // add task 1
    executor.tasks.push(Task::new()); // add task 2
    executor.tasks.push(Task::new()); // add task 3

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.run();
        }
    }
}
```

Nice. But these tasks don't do anything! Let's give each of them a name and make them print to the screen. This is the new entire code:

```rust
struct Task {
    name: String,
}

impl Task {
    fn new(name: String) -> Self { // new: name
        Self { name: name } // new: name
    }
    fn run(&mut self) {
        println!("Hi from task: {}", self.name); // new: print to screen
    }
}

struct Executor {
    tasks: Vec<Task>,
}

impl Executor {
    fn new() -> Self {
        Self { tasks: Vec::new() }
    }
}

fn main() {
    let mut executor = Executor::new();

    // new!
    executor.tasks.push(Task::new("task1".to_string())); // add task 1
    executor.tasks.push(Task::new("task2".to_string())); // add task 2
    executor.tasks.push(Task::new("task3".to_string())); // add task 3

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.run();
        }
    }
}
```

Wohoooo! Our code does *something*! But if you run the code above you'll see that, although it does something, it also never stops doing it. Here's a piece of the output:
```
Hi from task: task1
Hi from task: task2
Hi from task: task3
Hi from task: task1
Hi from task: task2
Hi from task: task3
Hi from task: task1
Hi from task: task2
Hi from task: task3
Hi from task: task1
Hi from task: task2
...
```

What is happening here? Our tasks seem to be running forever! Going back to our original idea: the plan was to have each task run for a bit, then circle back when we got to the end of the task queue, but we also need two more things: (1) a way for the task to signal it's done and (2) remove all tasks that are done before circling back to the begining of the queue.

For the first issue the solution is easy:
1. add a `bool` field called `done` to the `Task` struct
2. set the `done` field to `false` when creating the task
3. set the `done` field to `true` at the end of `run()`

**Obs**: I'll also add a simple method `is_done()` to check if the task is done (note that `is_done()` doesn't need `&mut self`, but rather `&self` since it just needs to read the struct, not mutate/change it).

**Obs2**: Don't stress too much about `&self` and `&mut self`, as these concepts are not too important for us.

```rust
struct Task {
    name: String,
    done: bool,
}

impl Task {
    fn new(name: String) -> Self {
        Self {
            name: name,
            done: false, // new!
        }
    }

    // new!
    fn is_done(&self) -> bool {
        self.done // returns self.done (equivalent to `return self.done;`, note that we don't put a ; at the end with this approach)
    }

    fn run(&mut self) {
        println!("Hi from task: {}", self.name);
        self.done = true; // new!
    }
}

// ... rest of the code
```

Now for our second issue, we can filter out of the vector all tasks that return `true` on `is_done()`, or rather, we'll `retain()` the ones that are *not* done:
```rust
// ... rest of the code

fn main() {
    let mut executor = Executor::new();

    executor.tasks.push(Task::new("task1".to_string())); // add task 1
    executor.tasks.push(Task::new("task2".to_string())); // add task 2
    executor.tasks.push(Task::new("task3".to_string())); // add task 3

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.run();
        }
        // new: clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```

Here's our new output:
```
Hi from task: task1
Hi from task: task2
Hi from task: task3
```

Eureka! Our executor runs the three tasks and stops running. Here's the full code for the curious:
```rust
struct Task {
    name: String,
    done: bool,
}

impl Task {
    fn new(name: String) -> Self {
        Self {
            name: name,
            done: false,
        }
    }

    fn is_done(&self) -> bool {
        self.done // returns self.done (equivalent to `return self.done;`, note that we don't put a ; at the end with this approach)
    }

    fn run(&mut self) {
        println!("Hi from task: {}", self.name);
        self.done = true;
    }
}

struct Executor {
    tasks: Vec<Task>,
}

impl Executor {
    fn new() -> Self {
        Self { tasks: Vec::new() }
    }
}

fn main() {
    let mut executor = Executor::new();

    executor.tasks.push(Task::new("task1".to_string())); // add task 1
    executor.tasks.push(Task::new("task2".to_string())); // add task 2
    executor.tasks.push(Task::new("task3".to_string())); // add task 3

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.run();
        }
        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```

Okay, but is this really what we wanted? Are we missing something? Think about it, then read on.


## Async

Let's review what we wanted in the begining:
> Ready? The idea is farily straightforward: we use a `queue` of tasks being run. We cycle through this list and run a part of each task for a little bit of time. Then, when we reach the end of the queue, we go back to the begining and do it all over again.
 We're missing the part where we run each task for **a little bit of time**. The way it is now, our implementation runs each task to completion, but that's not what we want. What if one of the tasks took a long time to run, while the next two were super quick? The two tasks would `starve` waiting for the processor to run them. To fix that, we'll first make our example a bit more complicated. Here are the new tasks, see if you can understand how they work (I'll explain below):


Let's add a field called `run_counter` that starts with zero and is incremented every time we call `run()`. When the counter is `100` and we call `run()`, the `done` field is set to `true` and the task returns. Then our executor will clean it up since it's `done`. What this effectively does is make every task print "Hi from task {name}" 100 times. For our purposes, this simulates the task being run partially every time we call `run()` on it:

```rust
struct Task {
    name: String,
    done: bool,
    run_counter: usize,
}

impl Task {
    fn new(name: String) -> Self {
        Self {
            name, // this is equivalent to `name: name`
            done: false,
            run_counter: 0,
        }
    }

    fn is_done(&self) -> bool {
        self.done
    }

    fn run(&mut self) {
        if self.run_counter == 100 {
            self.done = true;
            return;
        }
        self.run_counter += 1;
        println!("Hi from task: {}", self.name);
    }
}
```


But ideally we want our tasks to be general, that is, we want to create a task and pass a function that it'll execute, rather than having it hardcoded in the `run()` method. To do that, let's go back to our tooth brushing example from before. Let's say aunt Fefê brushes her teeth exactly 100 times, she's a very meticulous lady! Nathan, however, is not as clean, he brushes his teeth 30 times, and I'll brush mine 20 times. How would this look in terms of code?

First, we create a function that takes in the number of times a person brushes their teeth:

```rust
fn brush_teeth(times: usize) {
    for i in 0..times {
        println!("Brushing teeth {}", i);
    }
}
```

Now, we need a way to make our `run()` function execute the `brush_teeth()` function. There are two ways to do that in Rust: closures and function pointers. The former requires way too much dark magic (perhaps a topic for a future post?), so we'll go with the latter. Here's how we'd adjust our `Task` type to make this possible using function pointers:

```rust
struct Task {
    name: String,
    done: bool,
    max_runs: usize, // new!
    run_counter: usize,
    func: fn(usize), // new: f is a variable of type function pointer
}

impl Task {
    // f is a pointer to a fn that takes a usize as first (and only) argument and returns nothing
    fn new(name: String, max_runs: usize, f: fn(usize)) -> Self {
        Self {
            name,
            done: false,
            max_runs, // new!
            run_counter: 0,
            func: f, // new!
        }
    }

    fn is_done(&self) -> bool {
        self.done
    }

    fn run(&mut self) {
        if self.run_counter == self.max_runs { // new: run self.max_runs times
            self.done = true;
            return;
        }
        self.run_counter += 1;

        (self.func)(self.run_counter); // new!
                                       // old: println!("Hi from task: {}", self.name);
    }
}

fn brush_teeth(times: usize) {
    for i in 0..times {
        println!("Brushing teeth {}", i);
    }
}

struct Executor {
    tasks: Vec<Task>,
}

impl Executor {
    fn new() -> Self {
        Self { tasks: Vec::new() }
    }
}

fn main() {
    let mut executor = Executor::new();

    let gabe = Task::new("gabe".to_string(), 20, brush_teeth);
    let nathan = Task::new("nathan".to_string(), 30, brush_teeth);
    let fefe = Task::new("fefe".to_string(), 100, brush_teeth);

    executor.tasks.push(gabe);
    executor.tasks.push(nathan);
    executor.tasks.push(fefe);

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.run();
        }
        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```
Here's a summary of the changes:
1. Added a `func` field that will hold the function pointer to the `Task` struct.
2. Added a `max_runs` field that will hold the number of times to run `brush_teeth`.
3. Added an `f` and `max_runs` as inputs to the `new()` function.
4. Made `run()` call `(self.func)`, effectively executing the function that the task holds.
5. Finally, we changed the call to `executor.tasks.push(Task::new(...))` to include the `brush_teeth` function pointer and `max_runs`.


You might've noticed that this causes a new issue: aunt Fefê brushes her teeth 5050 times! Why is that? The `run()` function is the culprit:
```rust
fn run(&mut self) {
    if self.run_counter == self.max_runs {
        self.done = true;
        return;
    }
    self.run_counter += 1;

    (self.func)(self.run_counter); // new!
}
```
The issue is that `run()` calls `self.func`, which is `brush_teeth()`, `100` times, but each time `brush_teeth()` runs, it receives `run_counter` as argument. In the first call, `run_counter` is zero, so aunt Fefê will brush her teeth one time. The second call will have `run_counter == 2`, making her brush twice. At the end she'll brush her teeth `100` times in a single call! Adding all of those together gives us `5050` brushes when we wanted only `100`. The fix is as simple as:

```rust
fn run(&mut self) {
    if self.run_counter == self.max_runs {
        self.done = true;
        return;
    }
    self.run_counter += 1;

    (self.func)(1); // fixed!
}
```

This looks good, but our executor is still way too limited. For starters, our `Task` still very much depends on the `func` we're passing. What if we wanted to run different tasks? For instance, if I'm brushing my teeth (`brush_teeth(20)`) while Nathan does the dishes (`do_dishes(10, 20)`)? Now we have a problem, because these functions have different signatures (`brush_teeth(usize)` and `do_dishes(usize, usize)`, but our `Task` struct can only hold `fn(usize)`, not `fn(usize, usize)`). We need a more general and flexible approach. Enter Futures!


## Futures
Ever wonder what the future looks like? For us, it looks a lot like a Rust trait:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

But seriously now, what are we even looking at here? First of all, let's talk traits. Traits are what other programming languages like Java refer to as an Interface (although there might be slight differences between those definitions). Traits are a way to define general characteristics a type needs to fulfil in order to be considered "something". An example is in order: let's say a general characteristic a `Person` needs to fulfil in order to be considered a `Sibling` is the ability to `annoy` another `Sibling`. Here's how these would look in Rust:

```rust
trait Sibling {
    fn annoy(&mut self, other: &dyn Sibling);
}

struct Person {
    name: String,
}

impl Person {
    fn new(name: String) -> Self {
        Self { name }
    }
}

impl Sibling for Person {
    fn annoy(&mut self, other: &dyn Sibling) {
        println!("I am {} and I'm annoying you!", self.name);
    }
}

fn main() {
    let mut gabe = Person::new("gabriel".to_string());
    let nathan = Person::new("nathan".to_string());

    gabe.annoy(&nathan);
}
```

This is good because not only `Person`s can be `Siblings`, but any type that implements the `annoy` function. By our definition, any type `MyType` that *implements* all the functions specified in Sibling (`impl Sibling for MyType`) is a Sibling. This is very useful especially when we want to create functions that take or return different types. In the `Future` trait, we're basically saying: "Any type that implements the `poll` function is a Future". This is a great solution for our flexibility problem. With this, our executor can have a queue of `Future`s, and thus can hold many different types, provided they all implement the `Future` trait.


**Obs**: Another important thing to note is the `Output` type inside the trait. As every task/job/future will have a different output/return type, we need to specify what the output to our Future will be. This will make more sense as we `impl Future for Task`, so don't worry too much about it for now.


You may be wondering what the `poll` function is and what it does. Great thinking! Let's review that next.


## Poll
Remember that in our original tooth brushing implementation we had to `return` every time we wanted to tell the executor "I'm done doing *a little work*"? Well, `poll`ing is a better way to do exactly that. The idea is simple. On the executor side, we `poll` Futures. On the Future side, we return with one of two possible answers:
1. `Poll::Pending`: "I did some work but there's still more"
2. `Poll::Ready(T)`: "I did some work (or not) and I'm all done! Here's the output `T`"

`Poll` is an enum (not to be mistaken with the function `poll` that returns a variable of the type `enum Poll`). An enum is basically a struct that represent different options, in this case either `Ready` or `Pending`. For reference, here's the source code for the `Poll` enum:
```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

In case this is the first time you're hearing about async/futures/polling, don't stress too much about it. If this looks like a lot to take in, it's because it is! Take your time, come back later if you need to. Also, it might be a good idea to code along so you get a better feel of the issues and limitations we're facing. It might be hard to properly understand things like the need for `Poll` and `Future` (and even traits) without actually coding yourself.

\*clears throat\*

Let's continue.


## An async implementation of our executor

Now that we know that `Poll` and `Future` exist, let's try to use them to solve our problem


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
