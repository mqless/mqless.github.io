---
title: Introduction
permalink: /docs/home/
redirect_from: /docs/index.html
---

MQLess is a lightweight actor model framework on top of AWS Lambda.

## The Problem

Let's look at one of the most basics issue with writing concurrent software. What do you we do if two different functions want to update the same variable? If both run sequentially no issue (same thread). 
What if they run from different threads? The state gets corrupted. However, we all learned the solution. We use locks to solve that.
The problem is, locks are slow and art only a few people know how to master.

These realities result in a no-win situation:
* Without sufficient locks, the state gets corrupted.
* With many locks in place, performance suffers and very quickly leads to deadlocks.

Still, you might say, that not my problem, my system is stateless.
Not quite right, you just moved the state to other services, which are stateful.  
For example, how your software behaves if two (or more) operations try to update the same record in DB?
The solutions (optimistic && pessimistic concurrency) lead to the same problem as the example above, either to a corrupted state or slow software.

For some stateless applications, that is good enough, concurrency on same database row or logical entity is rare, and usually, they can live with optimistic concurrency.

For a high-performance low-latency application that is not enough and a better solution is needed.

Enter the actor model.

## Actor model

The actor model was proposed decades ago by Carl Hewitt as a way to handle parallel processing in a high-performance network — an environment that was not available at the time. Today, hardware and infrastructure capabilities have caught up with and exceeded Hewitt’s vision. Consequently, organizations building distributed systems with demanding requirements encounter challenges that cannot fully be solved with a traditional object-oriented programming (OOP) model, but that can benefit from the actor model.

Today, the actor model is not only recognized as a highly effective solution — it has been proven in production for some of the world’s most demanding applications. To highlight issues that the actor model addresses, this topic discusses the following mismatches between traditional programming assumptions and the reality of modern multi-threaded, multi-CPU architectures:

[Brian Storti](https://www.brianstorti.com/the-actor-model/) wrote an excellent 10-minutes article on actor model.













