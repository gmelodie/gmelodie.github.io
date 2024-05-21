+++
date = "2024-05-21"
title = "Total Madness #0: Locks"
series = ["Total Madness"]
+++

This is the first post in a series about my urges to find out how dark magic atually works in computers. You see, recently I've some more free time on my hands and decided to spend it to scratch the itches I've had for as long as I can code (writing an [OS in Rust](https://github.com/gmelodie/cruzos) for instance). So I decided to share some of the things I never really liked to assume, and that I've been discovering.

We'll talk about architectures, memory, concurrency, why we can't access address 0x0, how a computer boots, and much more. Buckle up, for we're going towards Total Madness.

# Episode 0: Locks

```rust
asdfsa
let var = 10;
loop {
  //comment
}
```
