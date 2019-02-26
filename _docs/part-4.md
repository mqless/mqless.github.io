---
title: Part 4 - Device Group Query
permalink: /docs/part-4/
---

## Introduction

In the previous part we created a naive implementation for querying the devices.
As you recall, the problem was that the device group actor was blocked until the process of querying the device is done.

The pattern to solve that in the actor model is to have an actor for querying the devices, that way, we release the device group to continue and process messages.
However, if we use the send API we still have to wait for the response and forward to the caller. 
We need a way to forward the handling of the message to another actor.

MQLess provide the forward method, which forward the message handling to another actor.

## Device group revisted

Let's go back to the device group, we need to update the request temperatures method to forward the message to the device group query actor.

```javascript
async function requestTemperatures(message) {
    const {address, endpoint, id} = message.header
    let {devices} = await getState(address)

    // We generate random address for the new device group query
    const recipient = uuid()

    // We forward the request with the list of devices
    await post(`${endpoint}/forward/${id}/DeviceGroupQuery/${recipient}`, {devices})

    return {}
}
```

For the query actor we generate a random address and it is only going to be used once.
As you can see, we are using a new package to generate the address, so install by `npm install --save uuid` and also import the function at the head of the file with `const uuid = require('uuid/v4')`.

To forward the message we call the forward method with the message id, function and address.
We also provide to the query actor the list of the devices. 

## Query Actor

The query actor implementation is the same as the previous implementation of the device group, except the list of the devices comes from the message and not from state.

```javascript
const {post} = require('axios')

async function deviceGroupQuery (message) {
    const {endpoint} = message.header
    const {devices} = message.payload

    // Send a ReadTemperature message to each of the actors concurrently and collecting the responses.
    const promises = devices.map(deviceId =>
        post(`${endpoint}/send/ReadTemperature/${deviceId}`).then(response => response.data))

    const temperatures = await Promise.all(promises)

    return {temperatures}
}

exports.deviceGroupQuery = deviceGroupQuery
```

Add the new actor to the template file under resources section:

```yaml
  DeviceGroupQuery:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/device-group-query.deviceGroupQuery
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess
```

## Testing

We can test the query actor by invoking the device group actor, same as in the previous part:

```shell
curl --data '{}' http://localhost:34543/send/RequestTemperatures/g1
```

## Summary

We reimplemented querying the devices without blocking the device group.
We still block a Lambda instance until all repsonses arrive, which might be a long time. 
When account is limited 1000 concurrent Lambda calls that might be a problem.

Above can be implemented in a completely asynchronous way, but it is out of scope for this tutorial.


