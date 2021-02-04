---
title: Analysis of a problem with async file IO in Rust
layout: post
---

Async IO on normal file handles do not work with `poll` or `select`, because files are considered "always ready". Consequently, developers use threads or processes to mimic such functionality to avoid blocking IO, which is widely adopted. Async runtimes for Rust, with no exception, inherited the design to enable async interactions with files. However, as threads running beneath the async surface, we cannot equal it with the "real" async code.

# Background

Several months ago, I was developing a toy project using Rust, and fanscinated by the newly stabalized async feature which I managed to squeeze into my project. At the same time, I came across the idea of FIFO/named pipe, which is a special file that behaves like a anonymous pipe:

- It can be created using `mkfifo` command
- It blocks until both ends are open, with one reading and one writing

Within the project, I decided to utilize a few named pipes to interact between different processes. Then the real problem surfaces.

# Problem

The part that caused me troubles is with the `select` function, normally used in async scenarios. Here is what `select` does in C, which is the same in Rust, and all other languages I believe:

> `select()` allows a program to monitor multiple file descriptors,
>        waiting until one or more of the file descriptors become "ready"
>        for some class of I/O operation (e.g., input possible).

With `select`, I was able to wait for several file descriptors, and abort the waiting whenever the first one of them becomes available. 

In the project, I meant to use `select` as a process interruption method. But the issue is, it is based on the interactions with named pipes. Based on the project design, named pipes are implemented for exchanging information between two processes. Whenever a piece of information arrives the receiver end, it must terminate all its ongoing task to work on the newly arrived job. Again, it is quite straightforward to use `select` here, that I only needed to put the two futures in the `select` block, and that will handle all the cancellation for me. Below is a piece of code to better illustrate the idea:

```rust
let mut input_text = input_pipe.receive().await;
loop {
  ...
  let next_input = input_pipe.receive().fuse();
  let task = work(input_text).fuse();
  ...
  select!(
    new_input = next_input => {
      //new input arrives first. Replace the input, and loop again
      ...
    }
    task_finished = task => {
      // task finished. Wait for the next input and move on
      ...
    }
  )
}
```

The code looks harmless and suppose to work, however, it is problematic actually. After running the code for a few minutes, I have noticed more than 200 threads spawned under the same process.

# Cause

So what is wrong here? The culprit is apprently the combination of thread-based async File IO and the named pipe. Whenever the code reaches the `select!`, the two futures (`task` and `next_input`) are expected to be polled alternately so that they can progress concurrently. Whenever one finishes, the other will not be polled and will be dropped/released. 

The expectation was simple, when `task` finishes, drop the `next_input`. However, the code above can only release the future object `next_input`, but not the thread running underneath it. Obviously, the code looks like it can do the job, since in Rust, when an object goes out of scope, it is dropped automatically. However, we also need to be careful about the concept again, that the async File IO is not really based on the `poll` function of the system, but rather it is based on threading. When we try to call `read()` function on a thread, and it will block the thread until it reads data back. This procedure is interminable externally without killing the whole process. As a result, whenever our code jumps to the branch when `task` finishes, it goes to the next loop with the previous `next_input` running and a new future waiting for the input (the comment of the `task` branch). This new future will take another thread from the designated thread pool (either in `async-std` or `tokio`). If this continues, there not will be enough threads for it and eventually it keeps spawning all the way to 200s. 

# Analysis

The problem apparently is caused by the errornous design. Giving the current design, async file io is based on threading with the lack of native support of `poll` on files. Threading on blockable read process is not dangerous alone, but wrapped with the skin of async and combined with async interruptible `select` api, it can be a big problem like **memory leak**. The most general case of memory leak is caused by forgetting to release a piece of memory before removing the pointer to it. Similarly here, the future `next_input` linked to a thread, but the thread was left out when future itself was released. So I think it can be better referred as **thread leak** if there is no such term before.

# Conclusion

I didn't solve the problem, but rather redesigned to circumvent it. The downside of this bug is that it forced me to rewrite a lot of code, but the upside is that it really sparked me to dig deeper into the designs of the async runtimes. For other developers who are using the same thread-based ~~fake~~ async file IOs, you should always be extra cautious.
