---
title: Analysis of a problem with async file IO in Rust
layout: post
---

Async IO on normal file handles do not work with `poll` or `select`, because files are considered "always ready". Consequently, developers use threads or processes to mimic such functionality to avoid blocking IO, which is widely adopted. Async runtimes for Rust, with no exception, inherited the design to enable async interactions with files. However, as threads running beneath the async surface, we cannot equal it with the "real" async code.

# Background

Several months ago, I was developing a toy project using Rust, and fanscinated by the newly stabalized async feature which I managed to squeeze into my project. At the same time, I came across the idea of FIFO/named pipe, which is a special file that behaves like a anonymous pipe:

- It can be created using `mkfifo` command
- It blocks until both ends are open, with one reading and one writing

Within the project, I decided to utilize a few named pipes to interact between different processes. Here the real problem surfaces.

# Problem

The section that caused me trouble is with the `select` function normally used in async scenarios. Here is what `select` does in C, which is the same in Rust, and all other languages I believe:

> `select()` allows a program to monitor multiple file descriptors,
>        waiting until one or more of the file descriptors become "ready"
>        for some class of I/O operation (e.g., input possible).

With `select`, I was able to wait for several file descriptors, and abort the wait and go ahead with the first one which becomes available. 

In the project, I was trying to build an inter-process one time use channel for data exchange. A dedicated process creates a named pipe with random name based on request, and sending data through the named pipe. The receiver process accepts the input data, and run certain time consuming transformation on it. However, if a new input comes in before finishing the transformation, it should abort and process on the new input. When the transformation finishes, the named pipe should be removed. Next time, the interaction will be through another named pipe with a new random name. Below is a piece of code to better illustrate the idea:

```rust
loop {
  ...
  let next_input = input_pipe.receive().fuse();
  let transform_text = transform(input_text).fuse();
	...
  select!(
    new_input = next_input => {
      //new input arrives first. Replace the input, and loop again
      ...
    }
    transform_finished = transform_text => {
      // transform finished. Move on
      ...
    }
  )
  // remove named pipe
}
```

The code looks harmless and suppose to work, however, it is problematic as hell. After running the code for a few minutes, it spawned 200 threads under the same process and became slow as a snail. 

# Cause

The culprit is apprently the combination of thread-based async File IO and the blocking named pipe. Whenever the code reaches the `select!`, the two futures (`transform_text` and `next_input`) are expected to be polled alternately. Whenever one future finishes, the other one is suppose to be terminated and released.

The expectation was simple, whenever `transform_text` finishes, the `next_input` will be dropped. Meanwhile, the reading from the named pipe should be terminated and the named pipe should be removed. However, the code above can only release the `next_input`, not the thread. That's because the reading process cannot be ceased externally, then the named pipe will consequently be unremovable. As a result, the code is able to continue executing, with the thread running and named pipe open. The next round, it waits on a new named pipe, and the same problem repeats. Eventually, the process drains out its file io thread pool and keeps spawning new threads all the way to 200.

# Analysis

The problem apparently is caused by the design for the edge case. Giving the current design, async file io is based on threading because there is not native support of `poll` on files. Threading on blockable read process is not dangerous alone, but wrapped with the skin of async and combined with async interruptible `select` api, it can be a big problem like **memory leak**. The most general case of memory leak is caused by forgetting to release a piece of memory before dropping the pointer to it. Similarly here, the future `next_input` linked to a thread, but the thread was left out when future was released when its task gets interrupted. Then the thread becomes inaccessible running behind the scenes. 

# Conclusion

I didn't solve the problem, but rather redesigned some process to circumvent it. The downside of this bug is that it forced me to rewrite a lot of code, but the upside is it really sparked me dig deeper into the designs of the async runtimes. For other developers who are using the same thread-based ~~fake~~ async file IOs, you should always be extra cautious.