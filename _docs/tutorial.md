---
title: Part 1 - The Example
permalink: /docs/part-1/
---

## Prerequisites
You installed MQLess locally together with DynamoDB.

## Introduction
In this tutorial we loosely follow the [Akka tutorial](https://doc.akka.io/docs/akka/current/guide/tutorial.html).

MQLess is much simpler than Akka, for better and for worst, some of the differences:
* MQLess doesn't has hierarchy or supervisors.
* MQLess doesn't manage the state (yet) - you need to manage the state elsewhere (like Redis or DynamoDB)
* MQLess actor always exists, virtually. You don't need to create or stop actors, just send them a message.
* MQLess is not a cluster, actors' messages are executed on AWS Lambda instances, MQLess is the router in front of Lambda.
* MQLess is language agnostic, you can write your actors in any language supported by AWS Lambda, and even use different languages within the same actor.

In this tutorial we will use NodeJS and Javascript.

## The Example
Internet of Things (IoT) system that reports temperature from sensor devices installed in customers’ homes. The target use case allows customers to log in and view the last reported temperature from different areas of their homes.

## Create a new project

Create a new nodejs project by run `npm init -y` in a new directory.
Currently the only dependency is the aws sdk for nodejs, so run `npm install --save aws-sdk`

## Actor State
As mentioned above, MQLess actors are stateless, so we need to save the actor state at an external storage, in this tutorial we will use DynamoDB.

> MQLess layer for AWS Lambda is on the roadmap, this will simplify the state management. Including caching of the state between function invokation on the Lambda instance.

For now, lets create a module for state, called `state.js` under the `src` directory.:

```javascript
const {config, DynamoDB} = require("aws-sdk")

if(process.env.AWS_SAM_LOCAL)
    config.update({endpoint: "http://dynamodb:8000"})

const docClient = new DynamoDB.DocumentClient()

const tableName = 'state'

async function getState (address) {
    const params = {
        TableName: tableName,
        Key: {address}
    }

    const data = await docClient.get (params).promise()
    return data.Item;
}

async function putState(address, state) {
    const item = Object.assign({address}, state)

    const params = {
        TableName: tableName,
        Item: item
    }


    await docClient.put(params).promise()
}

module.exports = {putState, getState}
```

## SAM

We are going to use [AWS SAM](https://github.com/awslabs/serverless-application-model) to deploy our functions to AWS and run them locally.

We need to add the state table to our template file.
So go on and create a `template.yaml` at the root of our project and paste the following:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: MQLess iot example

Resources:
  DynamoStatesTable:
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

The above code instruct AWS to create a Dynamodb table as part of deploying our package.

SAM doesn't create the table when running locally, so we need to create the table manually.
However you probably already did that if you follow the 'Install MQLess locally' guide.
Anyway, following is the snippet to install it locally.

```shell
aws dynamodb create-table --endpoint-url http://localhost:8000 \
  --table-name state \
  --attribute-definitions '[{"AttributeName": "address", "AttributeType": "S"}]' \
  --key-schema '[{"AttributeName": "address", "KeyType": "HASH"}]' \
  --provisioned-throughput '{"ReadCapacityUnits":1,"WriteCapacityUnits":1}'
  ```

## The Device

We are ready to write our device, our device should read and record the temperature.
We will save the temperature in the actor state. Create `device.js` under `src` directory.

```javascript
const {getState, putState} = require('./state')

async function readTemperature(message) {
    const {address} = message.header
    const state = await getState(address)

    if (!state)
        return {value: null}

    return {value: state.lastTemperatureReading}
}

async function recordTemperature(message) {
    const {address} = message.header
    const state = {lastTemperatureReading: message.payload.value}

    await putState(address, state)

    return {}
}

exports.readTemperature = readTemperature
exports.recordTemperature = recordTemperature
```

We also need to update our SAM template, add the following to the `template.yaml` file under `Resources` section:

```yaml
  ReadTemperature:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/device.readTemperature
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess

  RecordTemperature:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/device.recordTemperature
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess
```      

## Testing

We are almost ready to run our actor, first we need to build our SAM template with `sam build`.
To start SAM Lambda server  run `sam local start-lambda --docker-network mqless-local --host 0.0.0.0`.

> Add `sam local start-lambda --docker-network mqless-local --host 0.0.0.0` to package.json as your start script.

Now lets test our actor, first let's record a temperature, we will use 'A' as our actor address.

```shell
curl --data '{"value": "25"}' http://localhost:34543/send/RecordTemperature/A
```

We can now read the temperature with
```shell
curl --data '{}' http://localhost:34543/send/ReadTemperature/A
```

If you will try a different address we will get an empty result:
```shell
curl --data '{}' http://localhost:34543/send/ReadTemperature/A
```

## One Lambda per Actor

In the above exampe, for each function of the device (read and record) we created a different Lambda.
However, that is not always the best approach. Instead we can have one function for each actor, with the subject field distinguishing between the message types. 

The benefit for this will come later when we would want to cache the state between actor calls, batch messages or change behavior.
MQLess is agnostic to which type you use, for the tutorial, we will continue to use Lambda per each message the actor support.

## Summary

Of the first part of the tutorial we created the device actor, which record and read temperature.
We also created a SAM template and run the actor locally.

On the next part of the tutorial we will create the device group actor and see how it manage and query devices.
