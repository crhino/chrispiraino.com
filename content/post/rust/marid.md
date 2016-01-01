+++
author = "Chris Piraino"
comments = false
date = "2015-12-30T19:48:50-08:00"
draft = true
image = ""
menu = ""
share = true
slug = "marid"
tags = ["rust", "concurrency"]
title = "Marid - Task Parallelism and Process Management in Rust"

+++

When building distributed systems and the different components that make up these systems, a simple and composable way of managing the lifetime of the many different running processes becomes necessary. At Cloud Foundry, my day job, we tend to use [ifrit](https://github.com/tedsuo/ifrit) for managing multiple tasks within a single OS process. This has the advantage of being easily composable and a simple model for adding shutdown/failure behaviour. As an example, when shutting down a proxy server, one would want to allow all current connections to finish while not accepting any new connections. In the ifrit model, we can just wait for a specific signal to be sent to the proxy process and then proceed with draining. While of course one could implement other ways, the advantage that this model allows is that I can easily write unit tests against the component to ensure that draining is down correctly. Now consider adding a number of different tasks for this process, e.g. metrics reporting, some pipelining, and the composability really shines through.

Thus, in the spirit of ifrit, I have written Marid, a Rust library for managing multiple threads/tasks and exposing an easy-to-use signaling behaviour for shutdown or other purposes. The MVP for this library includes the `Runner` and `Process` traits, the `Composer` and `MaridProcess` structs, and the `FnRunner` type.

One of the more interesting parts of working on this crate is that this was really the first time I was dealing with trait objects. Trait objects are a basically a form of dynamic dispatch. Imagine a trait `Foo` and two different structs that implement it `BarStruct` and `QuxStruct`. A trait object for the trait `Foo` would allow our code to use instances of both `BarStruct` and `QuxStruct` interchangably during runtime. This of course comes at the cost of having to look up function pointers in the trait object's vtable. The Rust documentation has a great section on this for anyone who wants to learn more: https://doc.rust-lang.org/book/trait-objects.html. For marid, trait objects allow us to be polymorphic with respect to the types of the Runners that want to be run, and since marid is only concerned with the startup and shutdown of a process, the slight performance hit of using trait objects is not a concern.

I would love to see people kick the tires of this library and send me any feedback about what you think would be a useful addition or what is hard to use and not ergonomic. Also, a big shoutout to burntsushi's library [chan_signal](https://crates.io/crates/chan-signal), which is the current basis of much of marid.

//////////////////////////// Might get rid of this //////////////////////////////////////
A `Runner` is the specific task that a developer wants to run. Specifically, a `Runner` has a `setup` function and `run` function. The flow of a `Runner` involves calling `setup`, ensuring that all runtime initialization is successfully accomplished, and then `run` is called with a signal receiver. The `Runner` may run for an indefinite period of time, only returning when either a shutdown signal has been received or an error has occurred.

A `Process` represents a currently running `Runner`, and is essentially a client to interact with the running task. The process enables waiting for a `Runner` to be setup, waiting for a `Runner` to finish its work, and allows a user to signal the associated `Runner`.

The `Composer` and `MaridProcess` are two concrete implementations of a `Runner` and `Process`, respectively. The `Composer` takes a list of runnable tasks and spawns each of them in separate threads, coordinating signaling and shutdown behaviour between them. The `MaridProcess` is the marid crate implementation of a `Process`, and will run any runnable task.

The `FnRunner` allows `FnOnce` closures to be used as a `Runner` object, allowing for easy tinkerering and converting of other threaded implementations.

