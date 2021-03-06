---
permalink: /
layout: landing
---

## Overview

MQLess is a lightweight actor model framework on top of AWS Lambda.

The Actor Model provides a higher level of abstraction for writing concurrent and distributed systems. It alleviates the developer from having to deal with explicit locking and thread management, making it easier to write correct concurrent and parallel systems.

MQLess bring a new approach by running the actors on AWS Lambda instead of a cluster like Akka or Microsoft Orleans.

## Why

The current model of serverless is a stateless one. However, real world applications are not always stateless.
Actor model fits serverless nicely, so you benefit the serverless part but also benefit stateful part on AWS Lambda.

## How it works

MQLess is a router in front of AWS Lambda. 
When you send a message to an actor, MQLess queue the message to each actor mailbox and activate the actor on AWS Lambda.
When the actor is done processing the message, MQLess send the response back to the caller and process the next message on the mailbox.









