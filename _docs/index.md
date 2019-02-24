---
title: Introduction
permalink: /docs/home/
redirect_from: /docs/index.html
---

MQLess is a lightweight actor model framework on top of AWS Lambda.

## The Problem

Let's look at one of the most basics issue with writing concurrent software. What do we do if two different functions want to update the same variable from different threads? 
If we don't do anything the state gets corrupted. 
However, we all learned the solution. We use locks to solve that.
The problem is, locks are slow and complex (deadlocks) and art only a few people know how to master.

These realities result in a no-win situation:
* Without sufficient locks, the state gets corrupted.
* With many locks in place, performance suffers and very quickly leads to deadlocks.

Still, you might say, that not my problem, my system is stateless.
Not quite right, you just moved the state to other services, which are stateful.  
For example, how your software behaves if two (or more) operations try to update the same record in DB? Or two services try to update the same record in Redis?
The typical solutions, optimistic && pessimistic concurrency, lead to the same problem as the example above, either to a corrupted state or slow software.

For some stateless applications, that is good enough, concurrency on same database row or logical entity is rare, and usually, they can live with optimistic concurrency.

For a high-performance low-latency application that is not enough and a better solution is needed.

Enter the actor model.

## Actor model

The actor model was proposed decades ago by Carl Hewitt as a way to handle parallel processing in a high-performance network — an environment that was not available at the time. Today, hardware and infrastructure capabilities have caught up with and exceeded Hewitt’s vision. Consequently, organizations building distributed systems with demanding requirements encounter challenges that cannot fully be solved with a traditional object-oriented programming (OOP) model, but that can benefit from the actor model.

Actor model is a little similar to OOP, instead of a class we have an actor and instead of methods we have messages.















