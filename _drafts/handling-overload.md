---
layout: post
title:  "Breaking Production - Handling overload in the real world"
author: "Amanda Sposito"
categories:
  - elixir
  - littles-law
  - performance
---

When our application starts to handle more traffic things start to break. An application designed to handle [5.000 requests a day](https://blog.twitter.com/official/en_us/a/2010/measuring-tweets.html), it's not the same that can handle [500 million requests a day](https://blog.twitter.com/official/en_us/a/2014/the-2014-yearontwitter.html). Most of us should remember [Twitter's Fail Whale](https://thenextweb.com/twitter/2013/11/25/rip-fail-whale/) and the struggle to keep pace with its users and the sheer volume of tweets.

In a scenario like that, what should we do?

### Observability

As our software evolves and the traffic grows, chances are that we will need to improve performance of different parts of the system. To do that, we need to understand **what** the application is doing and explain **why** it behaves the way it does.

To achieve that we have three main pillars that can help us: Logs, Metrics and Traces.

##### Logs

This is normaly the easiest way to start, almost all languages alredy have a built-in logger that you can use. It logs events that happened in your application, and it's useful for debugging when something happens.

##### Metrics

In Elixir we have a lib called [Telemetry](https://github.com/beam-telemetry/telemetry) that unifies and standarizes the way we collect metrics.

It's important to define what matters to us on the operational side such as CPU usage, memory, database query times, cache hits and misses; and also what matters to the application domain, such as requests by second, users in real-time, etc. With that in hand we can start to measure our code.

##### Traces

It's a series of events that tracks the progression of a single request. Each unit of work in a trace is called a span, and a trace is a tree of spans. A span provides Request, Error and Duration metrics that can be used to debug availability as well as performance issues, as it gives us more visibility of what's happening in our system.

This can help us a lot, as it gives everyone the ability to understand and fix what's wrong in the system.

##### Using it all together

Now that we know the three pillars of observability, we can start monitoring our metrics and how the changes we do in our application impact the performance.

To operationalize those metrics by creating alerts based on what matters to us, and to monitor the variability of these metrics is what allows us to react to a not-so-good deployment, to a new feature that needs to be improved, or to a system degradation that could happen over time.

### Planning for overload

Performance is a feature, and there are type of systems that won't need to worry with scalability at all. But given that we are dealing with an application that needs scalability, and that we already have our metrics and the visibility of what's happening in the application. Where do we start?

![Calvin and Hobes, from Bill Waterson](/assets/images/breaking-production/calvin-and-hobbes-bridges-load-limit.jpg)

One way to guarantee that your system can withstand the load it was designed to handle, and, with time, scale to manage increased demand, it's to simulate high loads. But keep in mind that even if you can simulate high traffic in your tests, sometimes, this will not be even closer to the production patterns. If that's the case, load test is still useful to the system but there are other options that could help us identify the system behavior in production, such as [chaos engineering](https://en.wikipedia.org/wiki/Chaos_engineering) and [feature flags](https://en.wikipedia.org/wiki/Feature_toggle).

We need to measure the system throughput and latency. The first represents the number of items processed per unit of time and the former represents the time between making a request to when you see a result.

A balanced system under heavy load results in a constant throughput. A system under overload becomes slow to respond, and when the latency increases the throughput decreases and the queue increases. This could cause an outage.

Sometimes we need to put safeguards in place, ensuring users do not overflow the system with requests it can’t handle, this means that **we** will know how the system will fail, rather than let it explode.

### Little's Law

"In queueing theory, the long-term average number L of customers in a stationary system is equal to the long-term average effective arrival rate λ multiplied by the average time W that a customer spends in the system." - [Wikipedia](https://en.wikipedia.org/wiki/Little%27s_law)

What it means is that the queue length (L), is equal to the arrival rate (λ), mutiplied by the response time (W), and the formula is represented like <br /> `L = λW`.

A different way to think about it is that a queue is made by the number of items inside the system (working in progress), the arrival rate being the number of requests made to the system, and the response time being the mean time a item takes to leave the system.

We can tweak this formula to calculate the number of items inside the system (Working In Progress) `WIP = Throughput * Response time`, the response time `Response Time = WIP / Throughput`, or the system throughput `Throughput = WIP / Response time`.

##### But why does it matter?

Have you heard the term *overload* before? It means that your system is having a hard time doing its tasks because the number of items arriving is greater than the number of items leaving. Remember Twitter's whale?

Most of the time we cannot control the arrival rate in the system, sometimes it could be constant such as a black friday event, or it could happen through spikes of requests such as breaking news.

What we can control is the number of WIP items in the system by applying backpressure, we can also try to optimize the throughput, by removing bottlenecks from the request-processing path whenever possible.

If we don't pay attention to those two our systems can become unresponsive, overflowed with requests it can’t handle, degrading in performance or crashing all together. This is not good for you or your users.

How do we protect our systems then?

### Load regulation

Little's Law can help us identify what's the queue length desired to keep our system's health in good shape. To determine the number of requests the system can service we can use the WIP formula (*WIP = Throughput * Response time*). Meaning that any request that exceeds the number of working in progress items allowed cannot be immediately serviced and must be queued or rejected.

Let's think of one example to ilustrate this in real life.

#### Cartola FC

Here in Brazil, we have an app, named Cartola FC, to play football fantasy game. People can put together their teams and it's based on the Brazilian National Football league. After every match you will earn points based on how the players in your team are performing.

The app has 5.8 million active users, 2 billions of app opening, and 238k requests per second.
The traffic pattern is spiky, because people that have popular players on their teams, when this player scores a goal, all of them wants to check the points at the same time, and sometimes it has 1 million simultaneous accesses.

To prevent degradaded responses, after a certain threshhold, they implemented a message saying "full stadium", so the system don't get overflowed with requests it can answer. They did this because normally after a refresh those who were waiting now will be able to see the partial results, because people were only interested on checking it's points, but they don't stay for long.

Thinking about Little's Law, the arrival rate is sky high, but the audience doesn't stick around for too long during the games, by showing the message to the ones under the service threshold you assure that your users will get the response and the ones that aren't able yet, will probably get a response next time they refresh the page.

It's better to know how your system will fail, rather than let it explode. A degraded response in this case was a message telling to the user to try again, but it could be returning data from a cache. It depends on your business need how you will return a degraded response, but it's important to think about one.

### TL;DR;

We covered a lot of things in this blog post.

First we did a recap about what's observability and how you can use it to understand your system better.

>> double check this

By knowing how your system works, you can start planning for how it will behave under heavy load.

It's necessary to define what matters to you and what matters to your domain. Always keeping an eye into the metrics that are important, to know how the changes we implement in the code affects the performance. Those metrics together and the habilit to create alerts that matter based on those metrics, it's what allow us to react to a bad deploy, a feature that needs improvement or a system degradation.

To handle overload we need to identify first where to apply backpressure. It's important to not overengineer the system that does not need load regulation. Use your system observability to decide where the optimizations should be.

Ask yourself: how many simultaneous requests can go through the system before latency becomes too high or the app runs out of memory?

Put safeguards in place, ensuring users do not overflow the system with requests it can’t handle. Know how your system will fail, rather than let it explode, and plan degraded responses that make sense to your business domain.

I hope it helps!

---

**Related Links**

* [https://samuelmullen.com/articles/the-hows-whats-and-whys-of-elixir-telemetry/](https://samuelmullen.com/articles/the-hows-whats-and-whys-of-elixir-telemetry/)
* [https://docs.honeycomb.io/learning-about-observability/events-metrics-logs/](https://docs.honeycomb.io/learning-about-observability/events-metrics-logs/)
* [https://opentelemetry.io/docs/concepts/data-sources/](https://opentelemetry.io/docs/concepts/data-sources/)
* [https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581](https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581)
* pt-BR [https://hipsters.tech/tecnologia-no-cartola-fc-hipsters-ponto-tech-232/](https://hipsters.tech/tecnologia-no-cartola-fc-hipsters-ponto-tech-232/)
* [https://keathley.io/blog/regulator.html](https://keathley.io/blog/regulator.html)
* [Designing for Scalability with Erlang/OTP: Implement Robust, Fault-Tolerant Systems](https://www.amazon.com/Designing-Scalability-Erlang-OTP-Fault-Tolerant-ebook-dp-B01FRIM8OK/dp/B01FRIM8OK/ref=mt_other?_encoding=UTF8&me=&qid=)
