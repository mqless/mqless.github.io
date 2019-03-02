---
title: Install MQLess locally
permalink: /docs/install-locally/
---

In order to run MQLess locally, we first need a way to run Lambda locally.
For IOT example we would also need DynamoDB to store the actors' state.

## Prerequisites
Install nodejs, docker, [AWS Cli](https://aws.amazon.com/cli/) and [SAM Cli](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).

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

## Install MQLess

We are going to install MQLess as docker container as well.

```bash
 docker run -d -p 34543:34543 --restart always --network mqless-local mqless/mqless --aws-local http://192.168.0.1:3001
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
  const address = message.address;
  const item = Object.assign({address}, message.payload);

  const params = {
    TableName: "state",
    Item: item
  }

  await docClient.put (params).promise();

  return {payload: {}}
}

exports.get = async function (message) {
  const address = message.address;
  const params = {
    TableName: "state",
    Key: {address}
  }

  const data = await docClient.get (params).promise();

  return {payload: data.Item};
}
```

Our test app is a simple key-value store on top of Lambda and DynamoDB, nothing cool about that yet, however because each key is an instance of an actor all reads and writes will be serialized into a queue (the mailbox) and AWS Lambda will process them one by one. Different keys will process parallel of course.

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
      FunctionName: put
      Handler: test-actor.put
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess

  GetFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: get
      Handler: test-actor.get
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess

  DynamoStateTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: state
      AttributeDefinitions:
        - AttributeName: address
          AttributeType: S
      KeySchema:
        - AttributeName: address
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 1
        WriteCapacityUnits: 1
```

As you can notice we also added the state table to the SAM file. But when running locally we have to create the table manually, so, run the following:

```bash
aws dynamodb create-table --endpoint-url http://localhost:8000 \
  --table-name state \
  --attribute-definitions '[{"AttributeName": "address", "AttributeType": "S"}]' \
  --key-schema '[{"AttributeName": "address", "KeyType": "HASH"}]' \
  --provisioned-throughput '{"ReadCapacityUnits":1,"WriteCapacityUnits":1}'
```

We are almost ready. Let's build the SAM template and run the lambda server:
```bash
mkdir .aws-sam/build -p
sam build
sam local start-lambda --docker-network mqless-local --host 0.0.0.0
```

You can now test MQLess, run the following to put a value for actor "A"

```bash
curl --data '{"value": "Hello World"}' http://localhost:34543/request/put/A
```

and now get it:

```bash
curl --data '{}' http://localhost:34543/request/get/A
```

## Summary

You now have MQLess installed locally together with DynamoDB and SAM cli in order to invoke the functions locally.
We also created a small test app and used MQLess for the first time.
