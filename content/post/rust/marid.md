+++
author = "Chris Piraino"
comments = false
date = "2015-12-30T19:48:50-08:00"
draft = false
image = ""
menu = ""
share = true
slug = "marid"
tags = ["rust", "concurrency"]
title = "Marid - Task Parallelism and Process Management in Rust"

+++

When building distributed systems and the different components that make up these systems, a simple and composable way of managing the lifetime of the many different running processes becomes necessary. At Cloud Foundry, my day job, we tend to use [ifrit](https://github.com/tedsuo/ifrit) for managing multiple tasks within a single OS process. This has the advantage of being easily composable and a simple model for adding shutdown/failure behaviour. As an example, when shutting down a proxy server, one would want to allow all current connections to finish while not accepting any new connections. In the ifrit model, we can just wait for a specific signal to be sent to the proxy process and then proceed with draining. While of course one could implement other ways, the advantage that this model allows is that I can easily write unit tests against the component to ensure that draining is down correctly. Now consider adding a number of different tasks for this process, e.g. metrics reporting, some pipelining, and the composability really shines through.

### Marid

Thus, in the spirit of ifrit, I have written Marid, a Rust library for managing multiple threads/tasks and exposing an easy-to-use signaling behaviour for shutdown or other purposes. The MVP for this library includes the `Runner` and `Process` traits, the `Composer` and `MaridProcess` structs, and the `FnRunner` type. The current version allows anyone to define multiple `Runner` trait objects and run them concurrently using the `Composer` and `MaridProcess` structs.

This might look something like:

```rust
let server = Box::new(HTTPServer::new()) as Box<Runner + Send>;
let task_pool = Box::new(TaskPool::new()) as Box<Runner + Send>;

let composer = Composer::new(vec!(server, task_pool), Signal::INT);
let signals = vec!(Signal::INT, Signal::HUP);

let process = launch(composer, signals);
process.wait().unwrap();
```

What the above code is showing are two different structs being created and cast as a `Box<Runner + Send>` trait object. The `Send` trait bound enables the Composer struct to send the runners to other threads that it spins up, and the `Runner` bound allows us to use the structs as runners. Next, these two runner objects and a signal are used to create the `composer` variable. The signal passed into the composer's `new` method represents the shutdown signal to send to all runners when any of the runners has finished. In practice, what this means is that if the `task_pool` for some reason panicked or errored out, the HTTP server would be quickly shutdown as well. This is a much better failure mode than having some threads correctly working and others not working at all. Finally, we launch our runners and specify the signals that the threads should be notified of, blocking on the result of the process.

### Trait objects and signaling

One of the more interesting parts of working on this crate is that this was really the first time I was dealing with trait objects. Trait objects are a basically a form of dynamic dispatch. Imagine a trait `Foo` and two different structs that implement it `BarStruct` and `QuxStruct`. A trait object for the trait `Foo` would allow our code to use instances of both `BarStruct` and `QuxStruct` interchangably during runtime. This of course comes at the cost of having to look up function pointers in the trait object's vtable. The Rust documentation has a great section on this for anyone who wants to learn more: https://doc.rust-lang.org/book/trait-objects.html. For marid, trait objects allow us to be polymorphic with respect to the types of the Runners that want to be run, and since marid is only concerned with the startup and shutdown of a process, the slight performance hit of using trait objects is not a concern.

In addition, ensuring proper signal handling behavior is a very hard issue to solve correctly. Marid currently relies on burntsushi's [chan-signal](https://crates.io/crates/chan-signal/) and, like the `chan-signal` crate, it is necessary for the `launch` function to be called before any threads are spawned in order to correctly ensure that signals are handled properly. Since Marid handles spawning threads for the user, this should not be an undue burden.

I would love to see people kick the tires of this library and send me any feedback about what you think would be a useful addition or what is hard to use and not ergonomic. Documentation can be found [here](http://crhino.github.io/marid-docs/marid/index.html), and the repository is [here](https://github.com/crhino/marid).
