+++
public = "true"
date = "2024-07-10"
title = "Total Madness #1: Async/Await"
series = ["Total Madness"]
+++


In the last episode (which is also the first episode, lol) we explored a bit of the world of concurrency by talking about locks. I mean, we didn't use the word "concurrent" explicitly, but effectively that's what we were talking about.

> Concurrency means multiple computations are happening at the same time.
>
> [MIT](https://web.mit.edu/6.005/www/fa14/classes/17-concurrency/)


Today we'll explore a crucial concurrency model: async/await.
Oh and don't you think I forgot about the promise I made about async locks, they're coming in the next post. First, though, we need to understand why async/await is even a thing, and how it *actually* works.

# Episode 1: Async/Await

You might remember the gross example we used to illustrate what happens on a computer when different *tasks* are competing for resources. Here's a refresher:

> Two siblings want to use the toothbrush they share, but obviously only one of them can use it at a time. The childish, sibling-like thing to do is to say “I’m first!” before the other person. The rule is simple: whoever starts saying “I’m first” first, gets to use it.

Now the word *task* here is an important, deliberate choice. We're talking about theoretical concepts, not what one programming language calls `Task` (and another would call `Process`, or `Thread`, or anything else). For us now, a task is simply a sequence of things a computer will do like add 1 to a variable or load a file from disk. For example, `task1` could be updating a file while `task2` would be printing to the screen. Here are the rough steps each task would need to accomplish:
```
Task1
1. task1 loads file from disk to memory (RAM)
2. task1 writes to file in memory
3. task1 saves file from memory to disk

Task2
1. task2 loads matrix of pixels from disk
2. task2 updates matrix of pixels
3. task2 writes matrix of pixes to screen
```

Because we want our computers to do a lot of things at once but have limited processing units (in our theoretical computer, we only have one processing unit!), clever computer people created the concept of **multitasking**, where tasks run intermittently to give the users the *impression* that everything is happening at once. Behind the scenes, however, the computer jiggles tasks like crazy to create this illusion. In our `task1` and `task2` examples from before, the sequence of instructions running could look like this:


```
1. task1 loads file from disk to memory (RAM)
2. task2 loads matrix of pixels from disk
3. task1 writes to file in memory
4. task1 saves file from memory to disk
5. task2 updates matrix of pixels
6. task2 writes matrix of pixes to screen
```

As you can see, the tasks are broken into pieces and jiggled by the processing unit. How would you design an algorithm to do that given a list of tasks?


## Tasks
Ready? A first, naïve idea, is fairly straightforward: we use a `queue` of tasks being run. We cycle through this list and run a part of each task for a little bit of time. Then, when we reach the end of the queue, we go back to the beginning and do it all over again.

Seems simple enough, but a ton of issues can stem from this approach, as well as a ton of ways to solve them. Let's implement a simple task runner following this idea. In case you're new to the posts, the examples are in Rust, but I'll explain the non-trivial parts.

So I'm thinking that we'll eventually need to have a struct holding important information about our executor, but I have no idea what that is at this point, so I'll make an empty struct for now and focus on the methods I want it to have.

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

Okay, we have a struct, an associated `new()` function/method that returns a new instance of the struct, and a way to call it. The Rust compiler complains of course: "You're not doing anything!", it says. The compiler is right, so let's do something. But, before we do anything, let's also create a `Task` type that our executor will hold in the queue:

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
`run()` takes the `Task` it refers to as a mutable reference (hence the `&mut self` as the first argument), this is because it will need to change something in the `Task` struct when `run()`ing. To run our tasks, we'll add a `for` loop that goes from start to finish of the list, calls `run()` on each task, and do that for as long as we have tasks to run:


```rust
// ... rest of code

fn main() {
    let mut executor = Executor::new();

    // new!
    while executor.tasks.len() != 0 { // while there are taks at queue
        for task in executor.tasks.iter_mut() { // go over each task at queue
            task.run(); // run task
        }
    }
}
```

Here we used `iter_mut` to get an [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html) of mutable references (remember our `run()` method needs to be able to mutate the struct). Now we just need to create some tasks to add to our executor:

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

Wohoooo! Our code does *something*! Now the problem is that it never stops "*doing* something". Here's a part of the output:

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

What is happening here? Going back to our original idea: the plan was to have each task run for a bit, then circle back when we got to the end of the task queue, but we also need two more things: (1) a way for the task to signal it's done and (2) remove all tasks that are done before circling back to the begining of the queue.

For the first issue, the solution is easy:
1. add a `bool` field called `done` to the `Task` struct
2. set the `done` field to `false` when a task is first created (inside `new()`)
3. set the `done` field to `true` at the end of `run()`

**Obs**: I'll also add a simple method `is_done()` to check if the task is done (note that `is_done()` doesn't need `&mut self`, but rather `&self` since it just needs to read the struct, not mutate/change it).

**Obs2**: Don't stress too much about `&self` and `&mut self`, as these concepts are not too important to us.

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

Now for our second issue, we can remove all tasks that are done from the `tasks` vector. A task is `done` if `is_done()` returns `true`. Rather, we'll `retain()` all the tasks that return `false` on `is_done()`:

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
        // new: clean up tasks that are done (aka retain tasks that are not done)
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

Let's review what we wanted in the beginning:
> A first, naïve idea, is fairly straightforward: we use a `queue` of tasks being run. We cycle through this list and run a part of each task for a little bit of time. Then, when we reach the end of the queue, we go back to the beginning and do it all over again.

 We're missing the part where we run each task for **a little bit of time**. Our implementation runs each task to completion before moving on, but that's not what we want. What if one of the tasks took a long time to run, while the next two were super quick? The two tasks would **starve** waiting for the processor to run them. To fix that, we'll first make our example a bit more complicated. Here are the new tasks, see if you can understand how they work (I'll explain below):


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

We added a field called `run_counter` that starts with zero and is incremented every time we call `run()`. When the counter is `100` and we call `run()`, the `done` field is set to `true`, and the task returns. Then our executor will clean it up since `is_done() == true`. What this effectively does is make every task print "Hi from task {name}" 100 times. For our purposes, this simulates the task being run partially every time we call `run()` on it.


Ideally, however, we want our tasks to be general and flexible, that is, we want to create a task and pass a function that it'll execute, rather than having it hardcoded in the `run()` method. To do that, let's go back to our tooth-brushing example from the last post.


Let's say Aunt Fefê brushes her teeth exactly `100` times, she's a very meticulous lady! Nathan, however, is not as clean, he brushes his teeth `30` times, and I'll brush mine `20` times. How would this look in terms of code?

First, we create a function that takes in the number of times a person brushes their teeth:

```rust
fn brush_teeth(times: usize) {
    for i in 0..times {
        println!("Brushing teeth {}", i);
    }
}
```

Now, we to make our `run()` function execute the `brush_teeth()` function. There are two ways to do that in Rust: **closures** and **function pointers**. The former requires way too much magic (perhaps a topic for a future post?). Here's how we'd adjust our `Task` type to use function pointers:


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
Summary of the changes:
1. Added a `func` field that will hold the function pointer to the `Task` struct.
2. Added a `max_runs` field that will hold the number of times to run `brush_teeth`.
3. Added an `f` and `max_runs` as inputs to the `new()` function.
4. Made `run()` call `(self.func)`, effectively executing the function that the task holds.
5. Finally, we changed the call to `executor.tasks.push(Task::new(...))` to include the `brush_teeth` function pointer and `max_runs`.


You might've noticed that this causes a new issue: Aunt Fefê brushes her teeth 5050 times! Why is that? The `run()` function is the culprit:
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

The issue is that `run()` calls `self.func`, which is `brush_teeth()`, `100` times, but each time `brush_teeth()` runs, it receives `run_counter` as an argument. In the first call, `run_counter` is zero, so Aunt Fefê will brush her teeth one time. The second call will have `run_counter == 2`, making her brush twice. In the end, she'll brush her teeth `100` times in a single call! Adding all of those together gives us `5050` brushes when we wanted only `100` (you can thank [Gauss](https://letstalkscience.ca/educational-resources/backgrounders/gauss-summation) for the calculations here). The fix is as simple as:

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

This looks good, but our executor is still way too limited. For starters, our `Task` still very much depends on the `func` signature we're passing. What if we wanted to run different tasks? For instance, if I'm brushing my teeth (`brush_teeth(20)`) while Nathan does the dishes (`do_dishes(10, 20)`)? Now we have a problem, because these functions have different signatures (`brush_teeth(usize)` and `do_dishes(usize, usize)`), but our `Task` struct can only hold `fn(usize)`, not `fn(usize, usize)`, and certainly not *both*! We need an even more general and flexible approach. Enter Futures!

**Note:** This next section will be a bit Rust-specific, but bear with me as these concepts are important to really understand async. Also, we'd have to go through this language-specific part ragardless of the language we were using anyway.

## Futures
Ever wonder what the future looks like? Here it is:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

But what are we even looking at here? First of all, let's talk traits. Traits are what other programming languages like Java refer to as an **Interface** (although there might be slight differences between those definitions). In Rust, traits are a way to define general characteristics a type needs to fulfill in order to be considered "something".

An example is in order: let's say a general characteristic a `Person` needs to fulfill in order to be considered a `Sibling` is the ability to `annoy` another `Sibling`. This is how it would look in Rust:
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


This is good because `Person` is not the only type that can be `Sibling`, but any type that implements the `annoy` function. By our definition, any type `MyType` that *implements* all the functions specified in `Sibling` (`impl Sibling for MyType`) **is** a `Sibling`. Say we had an `Elephant` struct. As long as we corrently implement all the functions that `Sibling` requires, with an `impl Sibling for Elephant` clause, then the `Elephant` struct could be a `Sibling` as well!


This is very useful especially when we want to create functions that take or return different types. Remember our problem of `Task` receiving functions of both `fn(usize)` and `fn(usize, usize)`? We could just have it receive a `Future` and it'd be able to receive different types, as long as they're `Future`s.


Now let's go back to the `Future` trait:
```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

Here we're basically saying: "Any type that implements the `poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>` function is a `Future`".


**Obs**: Another important thing to note is the `Output` type inside the trait. As every task/job/future will have a different output/return type, we need to specify what the output of our Future will be. This will make more sense as we `impl Future for Task`, so don't worry too much about it for now.


You may be wondering what the `poll` function is and what it does. Great thinking! Let's review that next.


## Poll
Remember that in our original tooth brushing implementation we had to `return` every time we wanted to tell the executor "I'm done doing *a little work*"? Well, `poll`ing is a better way to do exactly that. The idea is simple:
- On the executor side we `poll` Futures;
- On the Future side we return one of two possible answers:
    1. `Poll::Pending`: "I did some work but there's still more"
    2. `Poll::Ready(T)`: "I did some work (or not) and I'm all done! Here's the output `T`"

`Poll` is an enum (not to be mistaken with the function `poll` that returns a variable of the type `enum Poll`). An enum is basically a struct that represents different options, in this case, either `Ready` or `Pending`. For reference, here's the source code for the `Poll` enum:
```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

This is a good time to stop and appreciate how much we've covered so far: a lot! Come back later if you need to. Also, it might be a good idea to code along. It can be hard to understand things like the need for `Poll` and `Future` (and even traits) without actually writing code.

\*clears throat\*

Let's continue.


## An async implementation of our executor

Now let's try to use `Poll` and `Future` to solve our problem. First, we'll change our `Task` struct to hold a `Future` instead of a function pointer.

```rust
pub struct Task {
    name: String,
    done: bool,
    future: Pin<Box<dyn Future<Output = String>>>,
}
```

And our `new()` function will take in a future as well:
```rust
pub fn new(name: String, future: impl Future<Output = String> + 'static) -> Self {
    Self {
        name,
        done: false,
        future: Box::pin(future),
    }
}
```
Again, we need to do that because a function pointer has a strict function signature (like `fn(usize)`, which *needs* to be a function that takes exactly one `usize` and returns **"nothing"**). In other words, all `Task`s will need to have the same function signature, while with `Future`s that's not the case.


**Obs**: Let's accept the concepts of **"nothing"**, `Pin`, `Box`, and the `dyn` keyword as things we need to have for our code to work. We can go into detail about why those are important in a later post.

Now we can write a `poll` method for our `Task` struct:
```rust
pub fn poll(&mut self) -> String {
    let binding = futures::task::noop_waker();
    let mut cx = Context::from_waker(&binding);

    match self.future.as_mut().poll(&mut cx) {
        Poll::Ready(output) => {
            self.done = true;
            output
        }
        Poll::Pending => "Task not finished".to_string(),
    }
}
```
Our `poll()` method in turn calls `poll()` on a mutable reference to our `future` (hence the `.as_mut()` there). Another thing to note is that the `poll()` method on `Future`s takes in a `Context`, but for now we're just using a placeholder from `noop_waker()` that holds no information whatsoever.


Now for the important part: we do a `match` (which is like a switch/case or an if/else clause, but much more powerful) and, if the return result of the `poll()` function is `Poll::Ready(output)` we set `self.done = true`, take that `output` and return it ourselves. Otherwise, if the return result is `Poll::Pending` we return a string saying we're not done.

**Obs:** We will not go into pattern matching and how this makes `match` incredible in this post, again, let's save that discussion for the future.


In our main function, things are pretty much the same:
```rust
fn main() {
    let mut executor = Executor::new();

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.poll();
        }
        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```

The only change was `task.run()` to `task.poll()`. But now you may be thinking: "Wait what??? We went through all that work for *this*?! This is stupid.". Well, not quite. You see, there's one thing missing in our `main()`, can you spot what it is? I'll give you a hint, it's actually three things.


We're missing... tasks! We never spawn any! How about we do that:
```rust
fn main() {
    let mut executor = Executor::new();

    let task1 = Task::new("first_task".to_string(), future);
    executor.tasks.push(task1);

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.poll();
        }
        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```

Okay, now we have a task on our executor, but trying to run this code gives us:
```bash
➜  cargo run
   Compiling episode-1-async v0.1.0
error[E0425]: cannot find value `future` in this scope
  --> src/main.rs:10:53
   |
10 |     let task1 = Task::new("first_task".to_string(), future);
   |                                                     ^^^^^^ not found in this scope

For more information about this error, try `rustc --explain E0425`.
error: could not compile `episode-1-async` (bin "default") due to 1 previous error
```

We never specify a `future`! Come to think of it, we haven't talked about how to **create** a future. Here's how: imagine we have a function we want the task to run. All we need to do is add the `async` keyword to that function and some `await` statements inside it to tell the compiler where it should return `Poll::Pending` (aka saying "I'm done working a little"), like so:

```rust
// new!
async fn brush_teeth(times: usize) -> String {
    for i in 0..times {
        println!("Brushing teeth {}", i);
    }
    return "Done".to_string();
}

fn main() {
    let mut executor = Executor::new();

    let future = brush_teeth(10); // new!
    let task1 = Task::new("first_task".to_string(), future);
    executor.tasks.push(task1);

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.poll();
        }
        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```


This code runs just fine, but we need a final touch. Can you figure out what it is? Hint: try to figure out what the output of the following code will be versus what it *should* be:

```rust
async fn brush_teeth(times: usize) -> String {
    for i in 0..times {
        println!("Brushing teeth {}", i);
    }
    return "Done".to_string();
}

fn main() {
    let mut executor = Executor::new();

    let future = brush_teeth(10);
    let task1 = Task::new("first_task".to_string(), future);
    executor.tasks.push(task1);

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            task.poll();
        }
        println!("Went through all tasks once"); // new!

        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```
The output is:
```
Brushing teeth 0
Brushing teeth 1
Brushing teeth 2
Brushing teeth 3
Brushing teeth 4
Brushing teeth 5
Brushing teeth 6
Brushing teeth 7
Brushing teeth 8
Brushing teeth 9
Went through all tasks once
```

But it should be:
```
Brushing teeth 0
Went through all tasks once
Brushing teeth 1
Went through all tasks once
Brushing teeth 2
Went through all tasks once
Brushing teeth 3
Went through all tasks once
Brushing teeth 4
Went through all tasks once
Brushing teeth 5
Went through all tasks once
Brushing teeth 6
Went through all tasks once
Brushing teeth 7
Went through all tasks once
Brushing teeth 8
Went through all tasks once
Brushing teeth 9
Went through all tasks once
```

One way to let our executor know we're done running a little is to use the `pending!()` macro from the `futures` crate. A crate is a library in Rust. A library is a collection of utilities (functions, structs, traits, enums, etc.) you can `use` in your code. Here's how we'll use this:


```rust
use futures::pending; // import the pending!() macro

async fn brush_teeth(times: usize) -> String {
    for i in 0..times {
        println!("Brushing teeth {}", i);
        pending!(); // new: I'm done with "a little work" (aka return Poll::Pending)
    }
    return "Done".to_string(); // I'm done with all the work (aka return Poll::Ready("Done")
}
```

The `use futures::pending` line imports the `pending!()` macro. A macro is used like a function, so we can see it as that (see [Appendix I](#appendix-i-rust-macros) for more about Rust macros). Another way of having `brush_teeth()` return after "a little" work is done would be to have a type `BrushTeethFuture` and implement the `Future` trait on it (`impl Future for BrushTeethFuture`) aka implement the `poll()` function which would return `Poll::Pending` after a single brush. In fact, that's a good exercise for the reader: try to implement `Future` for a `struct BrushTeethFuture`, put it inside our `Task`, and have the `Executor` run it and get the expected result.

Now let's unfold what the `pending!()` macro actually does. Here's the [source code](https://docs.rs/futures-util/0.3.30/src/futures_util/async_await/pending.rs.html) for it:

```rust
use core::pin::Pin;
use futures_core::future::Future;
use futures_core::task::{Context, Poll};

/// A macro which yields to the event loop once.
///
/// This is equivalent to returning [`Poll::Pending`](futures_core::task::Poll)
/// from a [`Future::poll`](futures_core::future::Future::poll) implementation.
/// Similarly, when using this macro, it must be ensured that [`wake`](std::task::Waker::wake)
/// is called somewhere when further progress can be made.
///
/// This macro is only usable inside of async functions, closures, and blocks.
/// It is also gated behind the `async-await` feature of this library, which is
/// activated by default.
#[macro_export]
macro_rules! pending {
    () => {
        $crate::__private::async_await::pending_once().await
    };
}

#[doc(hidden)]
pub fn pending_once() -> PendingOnce {
    PendingOnce { is_ready: false }
}

#[allow(missing_debug_implementations)]
#[doc(hidden)]
pub struct PendingOnce {
    is_ready: bool,
}

impl Future for PendingOnce {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<Self::Output> {
        if self.is_ready {
            Poll::Ready(())
        } else {
            self.is_ready = true;
            Poll::Pending
        }
    }
}
```
Phew! This may look like a mess, but it's actually pretty simple. Here's the breakdown:
1. There's a struct `PendingOnce` that holds an `is_ready` boolean field.
2. The `Future` trait is implemented for `PendingOnce`.
3. When `poll()` is called on `PendingOnce`:
    - If `is_ready == false`: set it to `true` and return `Poll::Pending`.
    - If `is_ready == true`: return `Poll::Ready(())`.
4. There's a function `pending_once()` that returns an instance of the `PendingOnce` struct with the `is_ready` field set to `false`. Much like our `Task::new()` and `Executor::new()` functions that create initialized structs.


What all of this effectively does is create a type (a `struct`) that returns `Poll::Pending` when first `poll()`ing, and after that it'll always return `Poll::Ready` (hence the name `PendingOnce`). The `pending!()` macro calls `pending_once().await`, which will make the future run. That's all `await` is.


And voilà! Here's our output now:
```
Brushing teeth 0
Went through all tasks once
Brushing teeth 1
Went through all tasks once
Brushing teeth 2
Went through all tasks once
Brushing teeth 3
Went through all tasks once
Brushing teeth 4
Went through all tasks once
Brushing teeth 5
Went through all tasks once
Brushing teeth 6
Went through all tasks once
Brushing teeth 7
Went through all tasks once
Brushing teeth 8
Went through all tasks once
Brushing teeth 9
Went through all tasks once
Went through all tasks once
```

As a final exercise, try to figure out why `Went through all tasks once` appears twice in the end.

Now of course there is one last thing (I promise) for us to do: solve our original toothbrushing problem. For that, all we need to do is add some tasks. Here's what that would look like:


```rust
fn main() {
    let mut executor = Executor::new();

    let task_gabe = Task::new("gabe".to_string(), brush_teeth(20));     // new: gabe + brush_teeth(20)
    let task_nat = Task::new("nat".to_string(), brush_teeth(30));       // new: nat + brush_teeth(30)
    let task_fefe = Task::new("fefe".to_string(), brush_teeth(100));    // new: fefe + brush_teeth(100)

    // new: push tasks
    executor.tasks.push(task_gabe);
    executor.tasks.push(task_nat);
    executor.tasks.push(task_fefe);

    while executor.tasks.len() != 0 {
        for task in executor.tasks.iter_mut() {
            print!("{}: ", task.name); // new: print task.name
            task.poll();
        }
        println!("--- Went through all tasks once"); // new: "--- " for better readability

        // clean up tasks that are done
        executor.tasks.retain(|task| !task.is_done());
    }
}
```

Here's the output for that code. Note that the executor cycles through the three tasks, but as long as `gabe` finishes, then only `nat` and `fefe` are alternating, until when there's only `fefe` left and it runs forever:


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
--- Went through all tasks once
gabe: Brushing teeth 6
nat: Brushing teeth 6
fefe: Brushing teeth 6
--- Went through all tasks once
gabe: Brushing teeth 7
nat: Brushing teeth 7
fefe: Brushing teeth 7
--- Went through all tasks once
gabe: Brushing teeth 8
nat: Brushing teeth 8
fefe: Brushing teeth 8
--- Went through all tasks once
gabe: Brushing teeth 9
nat: Brushing teeth 9
fefe: Brushing teeth 9
--- Went through all tasks once
gabe: Brushing teeth 10
nat: Brushing teeth 10
fefe: Brushing teeth 10
--- Went through all tasks once
gabe: Brushing teeth 11
nat: Brushing teeth 11
fefe: Brushing teeth 11
--- Went through all tasks once
gabe: Brushing teeth 12
nat: Brushing teeth 12
fefe: Brushing teeth 12
--- Went through all tasks once
gabe: Brushing teeth 13
nat: Brushing teeth 13
fefe: Brushing teeth 13
--- Went through all tasks once
gabe: Brushing teeth 14
nat: Brushing teeth 14
fefe: Brushing teeth 14
--- Went through all tasks once
gabe: Brushing teeth 15
nat: Brushing teeth 15
fefe: Brushing teeth 15
--- Went through all tasks once
gabe: Brushing teeth 16
nat: Brushing teeth 16
fefe: Brushing teeth 16
--- Went through all tasks once
gabe: Brushing teeth 17
nat: Brushing teeth 17
fefe: Brushing teeth 17
--- Went through all tasks once
gabe: Brushing teeth 18
nat: Brushing teeth 18
fefe: Brushing teeth 18
--- Went through all tasks once
gabe: Brushing teeth 19
nat: Brushing teeth 19
fefe: Brushing teeth 19
--- Went through all tasks once
gabe: nat: Brushing teeth 20
fefe: Brushing teeth 20
--- Went through all tasks once
nat: Brushing teeth 21
fefe: Brushing teeth 21
--- Went through all tasks once
nat: Brushing teeth 22
fefe: Brushing teeth 22
--- Went through all tasks once
nat: Brushing teeth 23
fefe: Brushing teeth 23
--- Went through all tasks once
nat: Brushing teeth 24
fefe: Brushing teeth 24
--- Went through all tasks once
nat: Brushing teeth 25
fefe: Brushing teeth 25
--- Went through all tasks once
nat: Brushing teeth 26
fefe: Brushing teeth 26
--- Went through all tasks once
nat: Brushing teeth 27
fefe: Brushing teeth 27
--- Went through all tasks once
nat: Brushing teeth 28
fefe: Brushing teeth 28
--- Went through all tasks once
nat: Brushing teeth 29
fefe: Brushing teeth 29
--- Went through all tasks once
nat: fefe: Brushing teeth 30
--- Went through all tasks once
fefe: Brushing teeth 31
--- Went through all tasks once
fefe: Brushing teeth 32
--- Went through all tasks once
...
```


## Appendix I: Rust Macros
Macros are processed before the actual code is compiled, and thus is an effective way to include and alter source code before compilation. On our `pending_once` example, the `pending!()` macro is defined as follows:

```rust
#[macro_export]
macro_rules! pending {
    () => {
        $crate::__private::async_await::pending_once().await
    };
}
```

This roughly means that when we call it in our code, the compile will replace `pending!()` with `futures::__private::async_await::pending_once().await`. So, when compiled, our code *actually* looks like this:

```rust
use futures::pending;

async fn brush_teeth(times: usize) -> String {
    for i in 0..times {
        println!("Brushing teeth {}", i);
        futures::__private::async_await::pending_once().await;
    }
    return "Done".to_string(); // I'm done with all the work
}
```


In this case, the macro is a mere and simple convenience, but they can be incredibly powerful. For more information check out [The Little Book of Rust Macros](https://veykril.github.io/tlborm/).

---
Thank you for reading! You can find the full source code for this episode at https://github.com/gmelodie/total-madness.

## References
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): I should note that I intentionally tried to mimic the writing style of this book, in which you make the reader stop and think before giving the answers. Huge shoutouts!
- [The Little Book of Rust Macros](https://veykril.github.io/tlborm/)

