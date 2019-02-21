---
title: Install MQLess locally
permalink: /docs/install-locally/
---

In order to run MQLess locally we first need a way to run Lambdas locally. 
For IOT example we would also need DynamoDB to store the actors' state.

## Prerequisites
Install docker, [AWS Cli](https://aws.amazon.com/cli/) and [SAM Cli](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).

On linux it is recommended to allow docker cli to run without sudo in order, follow this guide [https://docs.docker.com/install/linux/linux-postinstall/](https://docs.docker.com/install/linux/linux-postinstall/).

## Configure Docker Network

In order for the local Lambda Server and Dynamodb to talk to each other we need create a user-defined network.

```bash
docker network create mqless-local
```

## Install Dynamodb

We are going to use amazon docker for local dynamodb and configure it to use the network we just created.

```bash
docker run -d -p 8000:8000 --network mqless-local --name dynamodb amazon/dynamodb-local
```

## Install MQLess

We are going to install MQLess as docker container as well.



