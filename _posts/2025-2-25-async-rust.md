---
title: "Async Rust: When to Use It and When to Avoid It"
date: 2025-02-25 20:29:07 -0300
categories: [Blogging]
tags: [rust]
---

## Introduction

_"Asynchronous"_ (often abbreviated as _"async"_) is a widely used term in computer science, but its meaning varies depending on the context.

In this article, we will focus on a definition that captures its essence in software development:

<br />

> _Asynchronous programming is a technique that enables your program to start a potentially long-running task and still be able to be responsive to other events while that task runs, rather than having to wait until that task has finished. Once that task has finished, your program is presented with the result._

> [Introducing asynchronous JavaScript, mozilla.org documentation](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Async_JS/Introducing)

<br />

### The Rust Context: Why Async Programming Matters

You’ve already decided to use Rust—perhaps for its **performance**, **memory safety**, **low-level control**, or simply because you enjoy the language. Now, let’s assume you’re starting a **new Rust project**, and at some point, you realize the need for concurrency or parallelism. Naturally, you might ask:

<br />

> _"How do I achieve concurrency in Rust?"_

<br />

This question can also come up when working on an **existing Rust project** and looking for a library to handle a specific task. Suppose you find [`reqwest`](https://github.com/seanmonstar/reqwest), a popular Rust HTTP client, described as _"An easy and powerful Rust HTTP Client."_. Just to find out that in the project's **README** there is a line that states _"This asynchronous example uses Tokio."_, with the following example:

```rust
use reqwest::get;
use tokio;

#[tokio::main]
async fn main() {
    let body = get("https://www.rust-lang.org").await.unwrap().text().await.unwrap();
    println!("{}", body);
}
```

<br />

For a developer aiming to **minimize dependencies**, this immediately raises a question:

<br />

> _"What is Tokio, and why is it required just to make an HTTP request?"_

<br />

If you’ve just finished reading [Rust’s official book](https://doc.rust-lang.org/book), you probably haven’t even encountered _Tokio_ yet. However, **it is nearly impossible to explore Rust's ecosystem without running into it**.

<br />

This leads to two key questions:

1. **Why do some libraries, like `reqwest`, default to async execution and require a runtime like _Tokio_?**
2. **What exactly is an async runtime, and how does it work?**

<br />

### Concurrency vs. Parallelism: Key Differences

We’ve used the term _concurrency_ multiple times already, so let’s ensure we have a clear and shared understanding of its definition.

When developing software, the need for concurrent or parallel execution is likely to arise. These concepts are fundamental to modern programming, yet they are often misunderstood or used interchangeably. However, concurrency and parallelism are distinct.

To clarify the difference, let's refer to a well-known Stack Overflow answer:

<br />

> _**Concurrency** is when two or more tasks can start, run, and complete in overlapping time periods. It doesn't necessarily mean they will be executing simultaneously. For example, multitasking on a single-core machine._
>
> _**Parallelism** is when tasks literally run at the same time, such as on a multicore processor._

<span style="font-size: 0.8em">This explanation was provided by [Richie Hindle](https://stackoverflow.com/users/21886/richiehindle) in response to the question ["What is the difference between concurrency and parallelism?"](https://stackoverflow.com/questions/1050222/what-is-the-difference-between-concurrency-and-parallelism).</span>

<br />

To further clarify:

- **Parallelism is impossible on single-core processors, while concurrency is**.
- **All parallel tasks are concurrent, but not all concurrent tasks are parallel**.

<br />

## Why Async? Understanding Its Role

There are two main reasons for **asynchronous programming**:

1. **Handling High-Concurrency Workloads Efficiently**
   Modern applications rarely execute a single task at a time. They often handle **hundreds or thousands of concurrent tasks**. Async programming enables handling these tasks efficiently with fewer system resources than a thread-based model.

2. **Making Concurrency Easier to Manage**
   Managing many threads directly is **complex and error-prone**. Async programming avoids common pitfalls of **thread safety**, like **race conditions** and **deadlocks**, by ensuring that tasks explicitly yield control instead of competing for CPU time.

### Key Takeaways:

- Async programming is an alternative **concurrency model** that is **not based on threads**.
- It is often **more efficient** and **easier to manage** than traditional threading models.
- Async **enables concurrency** in **environments without native threading** support, such as WebAssembly (WASM) and embedded systems, making it a crucial tool in those domains.

<br />
<br />

While other _concurrent programming models_ exist—such as _coroutines_—this article focuses specifically on Rust. We’ll explore the concurrency models Rust provides: threads and async programming.

## Rust’s Built-in Async Constructs

Rust’s standard library includes built-in support for asynchronous programming, such as futures and the async/await syntax. However, these primitives alone are not enough to write fully asynchronous programs. The missing piece is what is called a _reactor_, which is essential for driving async I/O and timers.

While it is technically possible to implement a custom _reactor_ using only the standard library, this is complex and rarely practical for most applications. Instead, external async runtimes like _Tokio_ and _async-std_ provide both a custom executor and a reactor, offering a complete solution for efficient asynchronous programming in Rust.

As Rust’s documentation explains:

<br />

> _Futures has its own executor, but not its own reactor, so it does not support execution of async I/O or timer futures. For this reason, it’s not considered a full runtime. A common choice is to use utilities from futures with an executor from another crate._

> [Rust Async Book: The Futures Crate](https://conradludgate.com/posts/async)

<br />

In short, while Rust’s standard library lays the groundwork for async programming, a _full runtime_ is necessary to execute asynchronous tasks effectively. Libraries like _Tokio_ and _async-std_ fill this gap, making async programming in Rust more accessible and practical.

<br />

### Choosing an Async Runtime

There are multiple well-known async runtimes in Rust, including [Tokio](https://github.com/tokio-rs/tokio), [async-std](https://github.com/async-rs/async-std), and [smol](https://github.com/smol-rs/smol). Each has its strengths and design trade-offs. In this article, we will focus on **Tokio**, as it is the most widely used and supported async runtime in the Rust ecosystem.

<br />

## Rust's concurrency models

Rust provides two primary concurrency models: **threads** and **async**.

> _Note: It may provide more concurrency models in the near future._

<br />

Before diving into these models, it’s helpful to revisit how operating systems enable concurrent execution of multiple programs. Understanding this foundation will make it easier to grasp the similarities and differences between OS-level concurrency and concurrency within a single application.

Additionally, to set the stage for async programming, let’s first explore threads—how they work, their advantages, and their limitations. This will give us the necessary context to evaluate async’s strengths and trade-offs more effectively.

<br />

### CPU Scheduling: Preemptive vs. Cooperative

At the operating system level, **scheduling** determines how CPU time is distributed among processes. The two primary strategies are **preemptive** and **cooperative** scheduling.

- **Preemptive Scheduling**: The OS controls task switching, ensuring fairness and responsiveness. This is the standard approach in modern operating systems like Linux, Windows, and macOS.
- **Cooperative Scheduling**: Tasks voluntarily yield control to the OS. This method, used in early systems like classic Mac OS and Windows 3.x, depends on well-behaved programs. It is also common in some real-time computing [RTC](https://en.wikipedia.org/wiki/Real-time_computing#Hard) environments.

<br />

The key distinction is:

- In a **cooperative** system, a selfish program can refuse to yield, monopolizing the CPU.
- In a **preemptive** system, the OS can interrupt a task to maintain fairness.

<br />

Both of these strategies enable the OS to execute multiple tasks concurrently, a capability known as **multitasking**.

These OS scheduling strategies **should not be confused with asynchronous programming. Async programming operates at the application level** and typically uses cooperative scheduling within a program. Unlike OS-level scheduling, async runtimes (such as Rust’s Tokio) do not preempt tasks; instead, async tasks must explicitly yield using constructs like `await` (popularized by Javascript).

**Async programming** is a concurrency model that operates at the application level, allowing fine-grained control over task execution.

### Threads in Rust

Since most operating systems use **preemptive scheduling**, many programming languages have adopted a thread-based concurrency model, leveraging the OS’s built-in support for concurrency.

Since a process is an instance of a program running in the OS, and threads are the smallest unit of execution within a process, it is natural for languages to use threads as the primary concurrency mechanism. This is evident in languages like Java, where threads are the standard approach to concurrency.

However, working with multiple threads presents challenges, even in Rust. The key difficulty arises because **threads share the same memory space**, which introduces the risk of **data races** when multiple threads access and modify shared data simultaneously.

Rust addresses this challenge through its **ownership model** and strict compile-time checks, ensuring that data shared across threads is properly synchronized or safely transferred. This allows Rust to provide both safe and efficient concurrency without relying on runtime garbage collection or locks as a default mechanism.

<br />

> _**Thread safety** is the avoidance of data races—situations in which data are set to either correct or incorrect values, depending upon the order in which multiple threads access and modify the data._

> [Multithreaded Programming Guide, Chapter 6: Safe and Unsafe Interfaces](https://docs.oracle.com/cd/E19683-01/806-6867/6jfpgdco5/index.html#:~:text=Thread%20safety%20is%20the%20avoidance,access%20and%20modify%20the%20data)

<br />

Rust has built-in mechanisms to **enforce thread safety at compile-time**. Features like **ownership** and **borrowing** ensure that **data races are caught before execution**.

Below are examples from [The Rust Programming Language](https://doc.rust-lang.org/book/ch16-00-concurrency.html):

#### Example 1: Creating a New Thread

Here’s a basic example of spawning a new thread

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }
}
```

<br/>

When running this program, the output may look something like this:

```
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 2 from the main thread!
hi number 3 from the spawned thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
```

<br/>

Notice that the spawned thread does not complete its execution—it was supposed to print up to:

```
hi number 9 from the spawned thread!
```

<br/>

However, that does not happen. This is because the main thread terminates before the spawned thread has finished its work.

<br/>

- The main thread creates the new thread and continues executing its loop.
- It only iterates five times, sleeping for 1 millisecond per iteration.
- Meanwhile, the spawned thread is supposed to run nine iterations but does not get a chance to complete before the main thread exits.
- When the main thread finishes execution, the entire process terminates, including the spawned thread.

<br/>

To prevent the spawned thread from being prematurely terminated, we can use a **join handle**. This allows the main thread to wait for the spawned thread to finish before exiting:

<br/>

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();
}
```

<br/>

In the example above `join()` blocks the main thread until the spawned thread finishes. And now program produces the following output:

<br/>

```
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the main thread!
hi number 2 from the spawned thread!
hi number 3 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
```

<br/>

#### Example 2: Using Message Passing (Channels)

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {received}");
}
```

Rust’s **ownership model** and **type system** prevent common concurrency pitfalls, making it significantly easier to work with threads.

<br/>

## What Is an Asynchronous Runtime?

<br/>

### Asynchronous Programming

As mentioned at the beginning of this article, defining asynchronous programming precisely can be challenging. One definition you can find online comes from Wikipedia:

<br/>

> _Asynchrony, in computer programming, refers to the occurrence of events independent of the main program flow and ways to deal with such events. These may be "outside" events such as the arrival of signals, or actions instigated by a program that take place concurrently with program execution, without the program hanging to wait for results._

> [Asynchrony (computer programming)](<https://en.wikipedia.org/wiki/Asynchrony_(computer_programming)>)

<br/>

The key word in this definition is _independent._ In asynchronous programming, the program does not block while waiting for _certain types_ of results. Instead, it remains free to perform other tasks while waiting for those results. This is a powerful concept—it enables concurrency without relying on threads and eliminates concerns about _thread safety_.

The most common type of operations where this approach is useful are I/O operations, such as writting to a file or making network requests. In a traditional _synchronous_ model, programs execute sequentially—each operation runs in a linear fashion, and when an I/O operation occurs, the program simply waits for it to complete before moving on to the next task. This sequential nature is what makes synchronous programming easier to debug and understand.

In contrast, the _asynchronous_ model anticipates these _slow_ operations and uses constructs like `async/await` to manage them. This allows the developer to indicate when an I/O operation is occurring, so the program can intelligently pause that _task_ and move on to another _task_ from a _task pool._ In this context, a _task_ can be thought of as a unit of work—similar to a lightweight thread, often referred to as a _green thread_.

<br/>

> _In computer programming, a green thread is a thread that is scheduled by a runtime library or virtual machine (VM) instead of natively by the underlying operating system (OS). Green threads emulate multithreaded environments without relying on any native OS abilities, and they are managed in user space instead of kernel space, enabling them to work in environments that do not have native thread support._

> Taken from the [Green Thread article in Wikipedia](https://en.wikipedia.org/wiki/Green_thread)

<br/>

### The Runtime

This leads us to the concept of a _runtime_, which is where _Tokio_ comes in. Among several third-party libraries, _Tokio_ provides a custom runtime for executing asynchronous Rust code. However, _Tokio_ is more than just a runtime—it describes itself as a platform.

<br/>

> _Tokio is an event-driven, non-blocking I/O platform for writing asynchronous applications with the Rust programming language. At a high level, it provides a few major components:_
>
> - _A multithreaded, work-stealing-based task scheduler._
> - _A reactor backed by the operating system's event queue (epoll, kqueue, IOCP, etc.)._
> - _Asynchronous TCP and UDP sockets._

> [Tokio's GitHub Repository](https://github.com/tokio-rs/tokio?tab=readme-ov-file#overview)

<br/>

The Tokio runtime is simply additional software—built on top of Rust—that manages the execution of your program and assists in dividing it into _pausable tasks._ These tasks are defined using the `async` keyword and can voluntarily yield control using the `await` keyword.

Tasks are usually _paused_ by a developer using `.await` whenever they perform an operation that could block the OS thread running them, such as an I/O operation.

As mentioned earlier, the `async/await` syntax is built into Rust, making it possible to implement a custom runtime. However, what we refer to as a runtime is typically composed of at least two key components: an _executor_ and a _reactor_.

Rust’s standard library includes a built-in executor, **but it does not provide a reactor**. This is why Rust is often said to lack a full runtime, requiring a third-party library to fill that gap. In this case, we use Tokio as an example of a third-party library that serves as a full runtime for executing asynchronous Rust code.

<br/>

### Responsibilities of a Runtime

We will talk specifically about Tokio, but these concepts should apply to any other runtime.

We can identify at least two key responsibilities of the Tokio runtime:

**Task Scheduling**

- When a task invokes `await`, it _yields_ control to the runtime. The runtime is then responsible for deciding which other task to execute next.
- This role is typically played by what's called the _executor_ component of the runtime.

<br/>

**Managing Blocking Operations**

- The runtime provides primitives that allow programs to offload blocking operations to the OS and resume execution when those operations complete. This aspect is described in Tokio’s [README](https://github.com/tokio-rs/tokio) as _"Asynchronous TCP and UDP sockets."_ While that’s a specific example for networking, the runtime applies similar mechanisms for handling various types of I/O operations.
- This role is typically played by what's called the _reactor_ component of the runtime.
  <span style="font-size: 0.8em">_Note: How a runtime converts blocking operations into non-blocking operations depends on the OS and is beyond the scope of this article. In some cases, it may not even be possible due to OS limitations._</span>

A _runtime_ is simply a cohesive software component that integrates an _executor_ and a _reactor_ along with other utilities, allowing programs to be executed asynchronously.

<br/>

### Executors and Reactors

Now, let’s clarify the definitions of **Executor** and **Reactor** in more detail.

#### **Executor**

The _executor_ is responsible for running tasks—in Rust terms, ensuring that futures are driven to completion. It continuously monitors tasks to determine when they are ready to resume execution. However, rather than immediately executing a task as soon as it becomes ready, the executor schedules it for execution at an appropriate time.

In other words, the executor manages task scheduling and execution, ensuring that all scheduled tasks are eventually run. In Rust terms, this means the executor is responsible for polling futures.

<span style="font-size: 0.8em">Note: This is a simplified explanation. In reality, Tokio’s implementation is far more sophisticated than what can be summarized in a few sentences.</span>

#### **Reactor**

The term _reactor_ originates from the _reactor pattern_:

<br/>

> _The Reactor design pattern handles service requests that are delivered concurrently to an application by one or more clients. Each service in an application may consist of several methods and is represented by a separate event handler that is responsible for dispatching service-specific requests. Dispatching of event handlers is performed by an initiation dispatcher, which manages the registered event handlers. Demultiplexing of service requests is performed by a synchronous event demultiplexer._

> An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events\_, https://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf

<br/>

In simpler terms, the _reactor pattern_ is a type of _event-driven architecture_, where _event creators_ generate events, and _event consumers_ process them. This architecture relies on a central _event loop_ responsible for dispatching events to the appropriate consumers.

Within the runtime, the reactor helps optimize CPU usage (save CPU cycles). Instead of having the executor repeatedly check whether a task in the queue is ready to run, the reactor maintains its own queues and notifies the executor when a task is ready to be resumed.

In Rust terms, the reactor signals the executor when a future is ready to be polled again, allowing for more efficient task execution.

In Tokio:

- The **reactor** manages the OS's event queue.
- The **executor** handles task execution and scheduling.

<br/>

Within Tokio’s documentation, this is often referred to as the _I/O event loop_:

<br/>

> _An I/O event loop, called the driver, which drives I/O resources and dispatches I/O events to tasks that depend on them._

> [Tokio's Documentation](https://docs.rs/tokio/latest/tokio/runtime/index.html)

<br/>

Following there is an extremely simple example of how to setup a fresh rust
project leveraging Tokio to perform asynchronous HTTP requests.

<br/>

```rust
// First, create a fresh rust project
// $ `cargo new async_example`

// Now, you need to add the required dependencies to your Cargo.toml file.
// Run the following commands in your terminal:

// Add Tokio for async runtime
// $ `cargo add tokio --features="full"`

// Add reqwest for HTTP requests (with async support)
// $ `cargo add reqwest --features="json"`

// Add serde for JSON serialization/deserialization (optional but useful)
// $ `cargo add serde --features="derive"`

// Now, the Rust script:
// src/main.rs

// Then run
// $ `cargo run`

use reqwest::Client;
use tokio::task;
use std::error::Error;

// Define a type for the expected JSON response
#[derive(Debug, serde::Deserialize)]
struct Post {
    userId: u32,
    id: u32,
    title: String,
    body: String,
}

// Async function to fetch a post from the given URL
async fn fetch_post(client: &Client, url: &str) -> Result<Post, Box<dyn Error>> {
    let response = client.get(url).send().await?.json::<Post>().await?;
    Ok(response)
}

#[tokio::main] // This macro starts the Tokio runtime
async fn main() -> Result<(), Box<dyn Error>> {
    let client = Client::new(); // Create an HTTP client

    // Define a set of URLs to fetch data from
    let urls = vec![
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/2",
        "https://jsonplaceholder.typicode.com/posts/3",
    ];

    // Spawn multiple concurrent tasks to fetch the URLs
    let mut handles = Vec::new();
    for url in urls {
        let client = client.clone(); // Clone the client for each task
        let handle = task::spawn(async move {
            match fetch_post(&client, url).await {
                Ok(post) => println!("Fetched post: {:?}", post),
                Err(err) => eprintln!("Error fetching {}: {}", url, err),
            }
        });
        handles.push(handle);
    }

    // Wait for all tasks to complete
    for handle in handles {
        handle.await?; // Await each task and propagate errors if any
    }

    Ok(())
}
```

<br/>

## Asynchronous Programming and Threads

While it is possible to have an asynchronous runtime that does not use threads, that is not the case with Tokio. Tokio _does_ use threads—and for good reason. Most modern CPUs have multiple cores, so why use just one when you can leverage many? This means that asynchronous programming and multithreading can be combined.

However, this is a configurable aspect of Tokio. The runtime provides two scheduling modes:

- The **Multi-Thread Scheduler**
- The **Current-Thread Scheduler**

<br/>

By default, Tokio uses the _Multi-Thread Scheduler_, which enables task execution across multiple threads. When this configuration is enabled (which is the default), the runtime behaves as follows:

<br/>

> _It will start a worker thread for each CPU core available on the system. This tends to be the ideal configuration for most applications._

> [Tokio's Documentation](https://docs.rs/tokio/latest/tokio/runtime/index.html)

<br/>

## Using Synchronous and Asynchronous Code in the Same Rust Program

To add even more flexibility (or potential confusion, depending on how you see it), Rust allows mixing synchronous and asynchronous functions within the same program—and even calling them from one another. However, doing so requires caution, as it can be tricky to manage.

Tokio’s documentation provides a dedicated section on this topic, [Bridging with Sync Code](https://tokio.rs/tokio/topics/bridging). Since the details are covered there, we won't go into them here—just know that this interoperability is possible.

This property is not unique to Rust. For example, in the Ruby ecosystem, there is a gem called [async](https://github.com/socketry/async) that allows defining asynchronous methods in Ruby. These methods coexist seamlessly with the rest of the Ruby code—assuming the code is _thread safe_.

<br/>

## Conclusion: When to Use Async Rust and When to Avoid It

Choosing between synchronous and asynchronous Rust depends on your application’s requirements. While some developers advocate for always using async and others suggest avoiding it altogether, the most practical approach lies in evaluating the specific demands of your project.

### When to Use Async Rust

- **High-Concurrency I/O Workloads**: Async Rust is particularly beneficial for applications that must handle a high number of concurrent network requests or database queries efficiently.
- **Latency-Sensitive Services**: In scenarios where responsiveness matters (e.g., web servers, real-time data processing), async can help reduce blocking and improve throughput.
- **Environments with Limited Threading**: Async is useful in WebAssembly (WASM) and embedded systems, where system threads may not be available.

### When to Avoid Async Rust

- **CPU-Bound Tasks**: For intensive computations (e.g., cryptographic processing, image manipulation), a multi-threaded synchronous approach is generally better.
- **Simple or Low-Concurrency Applications**: If your application doesn’t require handling many simultaneous operations, async adds unnecessary complexity.
- **File System Operations**: As highlighted in Tokio’s documentation, Tokio does not provide any advantage for file reading and writing because most operating systems lack true asynchronous file system APIs

<br/>

Another important consideration is **binary size**—async Rust increases the final binary footprint due to the overhead of async state machines and runtime dependencies.

<br/>

In summary, **use async Rust when you need to manage many concurrent I/O-bound operations efficiently, particularly in network-heavy applications or constrained environments. Avoid it when working with CPU-heavy tasks, simple applications, or workloads where threading already provides an effective solution**.

<br/>

## Recommended Reading

- [The State of Async Rust: Runtimes](https://news.ycombinator.com/item?id=37639896)
- [Async Rust in Three Parts](https://jacko.io/async_intro.html)
- [Let's talk about this async](https://conradludgate.com/posts/async)
- [Asynchronous programming(C#)](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)

## References

- [The Rust Async Book](https://rust-lang.github.io/async-book/)
- [What is the difference between concurrency and parallelism?](https://stackoverflow.com/questions/1050222/what-is-the-difference-between-concurrency-and-parallelism) – Stack Overflow.
- [Async: What is Blocking?](https://ryhl.io/blog/async-what-is-blocking/) – A blog post by Ryan Levick.
- [Event Loop](https://en.wikipedia.org/wiki/Event_loop) – Wikipedia.
- [Thread Safety](https://docs.oracle.com/cd/E19683-01/806-6867/6jfpgdco5/index.html#:~:text=Thread%20safety%20is%20the%20avoidance,access%20and%20modify%20the%20data.) – _Multithreaded Programming Guide, Chapter 6: Safe and Unsafe Interfaces (Oracle Documentation)._

This blog post was originally published on [Wyeworks's](https://www.wyeworks.com/) blog, [Async Rust: When to Use It and When to Avoid It](https://www.wyeworks.com/blog/2025/02/25/async-rust-when-to-use-it-when-to-avoid-it/)
