---
title: Tutorial
permalink: /docs/tutorial/
---

## Prerequisites
You have an AWS account and install MQLess on an EC2 machine.

## Introduction
In this tutorial we loosely follow the [Akka tutorial](https://doc.akka.io/docs/akka/current/guide/tutorial.html).

MQLess is much simpler than Akka, for better and for worst, some of the differences:
* MQLess doesn't has hierarchy.
* MQLess is state less - you need to manage your state elsewhere (like Redis or DynamoDB)
* You don't need to create or stop actors, just send them a message. If it a new actor, MQLess will create it automatically. * * Actors are stopped automatically as well by AWS Lambda scheduler.
* MQLess in not a cluster, all actors are AWS Lambda instances,  MQLess is the router in front of Lambda.
* MQLess is language agnostic, you can write your actors in any language supported by AWS Lambda, and even use different languages within the same actor.

In this tutorial we will use NodeJS and Javascript.

## The Example
Internet of Things (IoT) system that reports temperature from sensor devices installed in customers’ homes. The target use case allows customers to log in and view the last reported temperature from different areas of their homes.

## Actor State
As mentioned above, MQLess actors are stateless, so we need to save the actor state at an external storage, in this example we will use DynamoDB.




