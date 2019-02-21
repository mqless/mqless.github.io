---
title: Install MQLess locally
permalink: /docs/install-locally/
---

In order to run MQLess locally we first need a way to run Lambdas locally. 
For IOT example we would also need DynamoDB to store the actors' state.

## Prerequisites
Install nodejs, docker, [AWS Cli](https://aws.amazon.com/cli/) and [SAM Cli](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).

On linux it is recommended to allow docker cli to run without sudo in order, follow this guide [https://docs.docker.com/install/linux/linux-postinstall/](https://docs.docker.com/install/linux/linux-postinstall/).

## Configure Docker Network

In order for the local Lambda Server and Dynamodb to talk to each other we need create a user-defined network.

```bash
docker network create mqless-local
```

## Install Dynamodb

We are going to use amazon docker for local dynamodb and configure it to use the network we just created.

```bash
docker run -d -p 8000:8000 --restart always --network mqless-local --name dynamodb amazon/dynamodb-local
```

## Install MQLess

We are going to install MQLess as docker container as well.

```bash
 docker run -p 34543:34543 --restart always --network host mqless/mqless --aws-local http://127.0.0.1:3001
 ```
 
 ## Testing
 
 We are going to create a small nodejs project to test everything:
 
 ```bash
 mkdir mqless-test-project
 cd mqless-test-project
 npm init -y
 npm install --save aws-sdk
 ```
 
Create `test-actor.js` and paste the following:

```js
const {config, DynamoDB} = require("aws-sdk");

if(process.env.AWS_SAM_LOCAL);
  config.update({endpoint: "http://dynamodb:8000"});
const docClient = new DynamoDB.DocumentClient();

exports.put = async function (message) {
  const routingKey = message.routing_key;
  const item = Object.assign({routingKey}, message.payload);
  
  const params = {
    TableName: "state",
    Item: item
  }
  
  await docClient.put (params).promise();
  
  return {}
}

exports.get = async function (message) {
  const routingKey = message.routing_key;
  const params = {
    TableName: "state",
    Key: {routingKey}
  }
  
  const data = await docClient.get (params);
  
  return {value: data.Item};
}
```

Our test app is a simple key-value store on top of Lambda and DynamoDB, nothing cool about that yet, however because each key is an instance of an actor all reads and writes will be serialized into a queue (the mailbox) and AWS Lambda will process them one by one. Diffrent keys will process parallely of course.

We are not done yet, we need to create the SAM file.
Create a `template.yaml` and paste the following:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: MQLess test project


Resources:
  PutFunction:
    Type: AWS::Serverless::Function
      Properties:
        Handler: test-actor.put
        Runtime: nodejs8.10
        Policies: AmazonDynamoDBFullAccess

  GetFunction:
    Type: AWS::Serverless::Function
      Properties:
        Handler: test-actor.get
        Runtime: nodejs8.10
        Policies: AmazonDynamoDBFullAccess
  
  DynamoStateTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: state
      AttributeDefinitions:
        - AttributeName: routingKey
          AttributeType: S
      KeySchema:
        - AttributeName: routingKey
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 5
        WriteCapacityUnits: 5
```

As you can notice we also added the state table to the SAM file. But when running locally we have to create the table manually, so, run the following:

```bash
aws dynamodb create-table --endpoint-url http://localhost:8000 \
  --table-name state \
  --attribute-definitions '[{"AttributeName": "routingKey", "AttributeType": "S"}]' \
  --key-schema '[{"AttributeName": "routingKey", "KeyType": "HASH"}]' \
  --provisioned-throughput '{"ReadCapacityUnits":1,"WriteCapacityUnits":1}'
```

We almost ready, lets build the SAM template and run the lambda server:
```bash
sam build
sam local start-lambda --docker-network mqless-local
```

You can now test MQLess, run the following to put a value for routingKey "A"

```bash
curl --data '"Hello World"' http://localhost:34543/send/A/put 
``` 

and now get it:

```bash
curl --data '{}' http://localhost:34543/send/A/get
```

## Summary

You know have MQLess installed locally together with DynamoDB and SAM cli in order to invoke the functions locally.
We also created a small test app and used MQLess for the first time
