---
title: Introduction
permalink: /docs/home/
redirect_from: /docs/index.html
---

MQLess is a lightweight actor model framework on top of AWS Lambda.

## The Problem

Let's look at one of the most basics issue with writing concurrent software. What do you we do if two different functions want to update the same state? If both run sequentially no issue (same thread). 
However, what if they run from a different thread? The state gets corrupted. 
However, we all know the solution, we use locks to solve that.
The problem is, locks are slow and art only a few people know how to master.

These realities result in a no-win situation:
* Without sufficient locks, the state gets corrupted.
* With many locks in place, performance suffers and very quickly leads to deadlocks.

However, you might say, that not my problem, I'm stateless.
Not quite right, you just moved the state to other services, which are stateful.  
For example, how your software behaves if two (or more) operations try to update the same record in DB?
The solutions (optimistic && pessimistic concurrency) lead to the same problem as the example above, either to a corrupted state or a slow system.

Enter the actor model.

## Actor model

