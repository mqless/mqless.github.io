---
title: Install MQLess locally
permalink: /docs/install-locally/
---

In order to run MQLess locally, we first need a way to run Lambda locally.
For IOT example we would also need DynamoDB to store the actors' state.

## Prerequisites
Install [NodeJS](https://nodejs.org/en/download/), [Docker](https://docs.docker.com/install/), [AWS Cli](https://aws.amazon.com/cli/) and [SAM Cli](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).

On linux, it is recommended to allow docker cli to run without sudo, follow this guide [https://docs.docker.com/install/linux/linux-postinstall/](https://docs.docker.com/install/linux/linux-postinstall/).

## Configure Docker Network

In order for the local Lambda instance and Dynamodb to talk to each other, we need to create a user-defined network.

```bash
docker network create mqless-local --subnet 192.168.0.0/24 --gateway 192.168.0.1
```

## Install Dynamodb

We are going to use amazon docker image for local dynamodb and configure it to use the network we just created.

```bash
docker run -d -p 8000:8000 --restart always --network mqless-local --name dynamodb amazon/dynamodb-local
```

## State Table

It is recommended to store the actors' state in Dyanmodb, as we do in the tutorial.
Run the following to create the table locally:

```bash
aws dynamodb create-table --endpoint-url http://localhost:8000 \
  --table-name state \
  --attribute-definitions '[{"AttributeName": "address", "AttributeType": "S"}]' \
  --key-schema '[{"AttributeName": "address", "KeyType": "HASH"}]' \
  --provisioned-throughput '{"ReadCapacityUnits":1,"WriteCapacityUnits":1}'
```

## Install MQLess

We are going to install MQLess as docker container as well.

```bash
 docker run -d -p 34543:34543 --restart always --name mqless --network mqless-local mqless/mqless --aws-local http://192.168.0.1:3001
```

## Summary

You now have MQLess installed locally together with DynamoDB and SAM cli.
You are ready to develop your MQLess applicaiton and test it locally.
