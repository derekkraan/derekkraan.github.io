---
layout: post
title: "What makes Sidekiq faster than Resque and DelayedJob?"
date: 2016-05-15 09:23:00 +0200
categories: background-jobs programming
---

I was involved in a short discussion on Reddit recently where someone asserted that Sidekiq would *not* be faster under IO-bound workloads. Googling, I wasn't able to find any blogposts that explained the conditions under which Sidekiq would outperform Resque and DelayedJob. This is that blogpost.

Inventory
===

First let's talk briefly about the resources a computer has. It's very helpful to know which of these resources are being maxed out first, as this will determine whether Sidekiq can make a difference or not. A modern, connected computer (like your server) has three basic resources: CPU, memory, and IO. By CPU I mean "all processing power", by memory I mean "temporary (RAM) and persisted (SSD/HDD) memory" and by IO I mean the computer's network connection to the outside world.

CPU Bound
===

Let's say you want to improve processing of your workload. This means: processing the same workload in less time (or a higher workload in the same amount of time, etc). It's important to know which of CPU, memory, or IO your workload is exhausting first. What does it mean when I say "CPU bound"? That simply means that our workload is using the maximum amount of available processing power before it has a chance to exhaust the other two resources: memory and IO.

If your workload is CPU bound, you basically have two options to improve the situation: add more CPU, or reduce the amount of CPU that you need (through optimization). The same holds true for memory-bound and IO-bound workloads. It's important to note that whenever you change the situation, you must re-assess. If your workload was CPU bound, and you added a whole bunch of CPU power to your server (or optimized your workload), it could now be memory-bound or IO-bound, so at a certain point, adding more CPU power (or optimizing CPU usage) will not deliver any more gains.

How is your workload bounded?
===

It's important to characterize your own workload:

* **CPU-bound**: CPU is pegged at 100% under load.
* **IO-bound**: Network connection is saturated under load OR network-connected services are being maxed out.
* **Memory-bound**: Memory is saturated, but neither CPU nor network are saturated.

How does Sidekiq help?
===

Sidekiq can help with two of these problems compared with Resque and DelayedJob: CPU-bound and memory-bound workloads.

Let's start with memory-bound workloads. Sidekiq uses a threaded model, while Resque and DelayedJob both use a forking model. When you fork in ruby, the entire memory of the process is duplicated. Threads, on the other hand, share memory. So assuming that you will be using more than 1 worker per machine, Sidekiq will allow you to dramatically reduce memory usage. If your workload is memory-bound, then this will allow you to run more workers per machine and better utilize the machine's other resources: CPU and network.

If your workload is CPU-bound, Sidekiq can also help with that. In addition to Sidekiq being threaded, it has also been optimized to death for CPU. The overhead per job has been made as small as possible. This comes with a caveat: if you have long-running CPU-bound jobs, then you won't notice a difference with Sidekiq. This is because the per-job improvement is very small compared to the total time spent processing a particular job. If, on the other hand, you have many small jobs (say, for example, many of your jobs perform a quick check and then exit), then you will most definitely notice a difference with Sidekiq.

Disadvantages of Sidekiq
===

Sidekiq is not a 100% drop-in replacement for Resque. Since Sidekiq uses threads (and therefore shares memory between threads), your codebase, or at least the parts being used in your background jobs, will need to be threadsafe. You have to consider not only your own code, but also the code of any gems you might be using. I know what you're thinking: what, there are gems that are not threadsafe? Yes, there are gems that are not threadsafe. Threading is still enough of a novelty in the ruby world that some gem maintainers have not built their gems with threading in mind.

Also, Sidekiq requires Redis. This is not a disadvantage vis-a-vis Resque, since Resque also uses Redis, but compared with DelayedJob, it does mean installing and maintaining another service on your production servers.

When can Sidekiq help?
===

To summarize, Sidekiq can help in the following situations:

- your total number of workers is memory-limited (and your workload is not consuming 100% CPU or saturating your network connection).
- your CPU usage hits 100% under load and you have many short-running jobs instead of few long-running jobs.

How much faster?
===

On the [Sidekiq FAQ](https://github.com/mperham/sidekiq/wiki/FAQ#what-kind-of-performance-can-i-expect-to-see-with-sidekiq), Mike Perham claims that you will usually see an order of magnitude improvement. This doesn't hold in certain situations, of course (if your workload is already CPU-bound with long-running jobs), but in other situations (short-running jobs, especially if your workload is not yet CPU-bound), Sidekiq really shines.
