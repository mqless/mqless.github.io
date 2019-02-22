---
title: Tutorial
permalink: /docs/tutorial/
---

## Prerequisites
You have an AWS account and install MQLess on an EC2 machine or install MQLess locally together with DynamoDB.

## Introduction
In this tutorial we loosely follow the [Akka tutorial](https://doc.akka.io/docs/akka/current/guide/tutorial.html).

MQLess is much simpler than Akka, for better and for worst, some of the differences:
* MQLess doesn't has hierarchy.
* MQLess is state less - you need to manage your state elsewhere (like Redis or DynamoDB)
* MQLess actor always exists, virtually. You don't need to create or stop actors, just send them a message.
* MQLess in not a cluster, all actors are AWS Lambda instances, MQLess is the router in front of Lambda.
* MQLess is language agnostic, you can write your actors in any language supported by AWS Lambda, and even use different languages within the same actor.

In this tutorial we will use NodeJS and Javascript.

## The Example
Internet of Things (IoT) system that reports temperature from sensor devices installed in customers’ homes. The target use case allows customers to log in and view the last reported temperature from different areas of their homes.

## Create a new project

Create a new nodejs project by run `npm init -y` in a new directory.
Currently the only dependency is the aws sdk for nodejs, so run `npm install --save aws-sdk`

## Actor State
As mentioned above, MQLess actors are stateless, so we need to save the actor state at an external storage, in this example we will use DynamoDB.

> MQLess layer for AWS Lambda is on our roadmap, this will simplify the state management. Including caching of the state between lambda invokes on the Lambda instance.

For now, lets create a module for state, called `state.js` under the `src` directory.:

```javascript
const {config, DynamoDB} = require("aws-sdk")

if(process.env.AWS_SAM_LOCAL)
    config.update({endpoint: "http://dynamodb:8000"})

const docClient = new DynamoDB.DocumentClient()
const tableName = 'states'

async function getState (routingKey) {
    const params = {
        TableName: tableName,
        Key: {"routingKey": routingKey}
    }

    const data = await docClient.get (params).promise()
    return data.Item;
}

async function putState(routingKey, state) {
    const item = Object.assign({"routingKey": routingKey}, state)

    const params = {
        TableName: tableName,
        Item: item
    }

    await docClient.put(params).promise()
}

module.exports = {putState, getState}
```

## SAM

We are going to use [AWS SAM](https://github.com/awslabs/serverless-application-model) to deploy our functions to AWS.
We are also going to use it to run our example locally.

We already need to create one resource to our template file for the new DynamoDB table we are using in the state module.
So go on and create a `template.yaml` at the root of our project and paste the following:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: MQLess iot example

DynamoStatesTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: States
      AttributeDefinitions:
        - AttributeName: routingKey
          AttributeType: S
      KeySchema:
        - AttributeName: routingKey
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 1
        WriteCapacityUnits: 1
```

The above code instruct AWS to create a Dynamodb table as part of deploying our package.

If you want to run the project locally we need to create the table manually, but you probably already did that if you follow the 'Install MQLess locally' guide, anyway, following it the snippet to install it locally.

```shell
aws dynamodb create-table --endpoint-url http://localhost:8000 \
  --table-name state \
  --attribute-definitions '[{"AttributeName": "routingKey", "AttributeType": "S"}]' \
  --key-schema '[{"AttributeName": "routingKey", "KeyType": "HASH"}]' \
  --provisioned-throughput '{"ReadCapacityUnits":1,"WriteCapacityUnits":1}'
  ```
  








