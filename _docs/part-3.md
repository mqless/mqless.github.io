---
title: Part 3 - Device Group
permalink: /docs/part-3/
---

## Introduction

In this part of the guide we will create the device group actor.
Device group is a collection of multiple devices, usually represent a home.
Device group actor main job is to query all of the devices in the group and return the list of temperatures to the user.
In order to do that the actor first need to know which devices are part of the group, so a device first need to register with the device group.

## Register

The registration protocol using MQLess is very simple, because actors in MQLess are always alive, we don't really need to create the device actor, we only need to add the new device id to the list of devices managed by the group.

Following is the code for `src/device-group.js`:

```javascript
const {getState, putState} = require('./state')

async function registerDevice(message) {
    const {address} = message.header
    const {deviceId} = message.payload

    let state = await getState(address)
    if (!state)
        state = {devices: []}

    state.devices.push(deviceId)
    await putState(address, state)

    return {
      payload: {
        deviceAddress: deviceId
      }
    }
}

exports.registerDevice = registerDevice
```

When the register function is being called we add the device id to the list of the devices managed by the actor.
However, more is happening here and we can begin see the benefit of the actor model.
What would happen if two devices would register at the exact same time without MQLess? Chances are, only one device will get registered.
However, with MQLess and actor model, the problem is solved, because the register messages will be handled one by one and both devices will be added to the list of devices.

We need to update our SAM template as well, add the following to `template.yaml` under the resources section:

```yaml
  RegisterDevice:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: device-group-register
      Handler: src/device-group.registerDevice
      Runtime: nodejs8.10
      Policies:
        - AmazonDynamoDBFullAccess
        - AWSLambdaVPCAccessExecutionRole
```

## Reading Temperatures

We would need the axios package in order to make http requests, run `npm install --save axios`.

Reading temperatures is a little more complicated than register a device.
In-order to read temperatures from the devices in the group we need to send them all the read temperature message.

We will use MQLess HTTP API. MQLess provide the endpoint in the header of the message, so you don't need any configuration and it works both locally and on AWS.

We will start with a naive implementation for reading the temperatures. Sending the read temperature message to each of the devices and wait for responses from all of them.

```javascript
const {post} = require('axios')

async function readTemperatures(message) {
    const {address, endpoint} = message.header
    let state = await getState(address)

    if (!state)
        return {temperatures: []}

    // Send a device-read-temperature message to each of the actors concurrently and collecting the responses.
    const promises = state.devices.map(deviceId =>
        post(`${endpoint}/request/device-read-temperature/${deviceId}`).then(response => response.data))

    const temperatures = await Promise.all(promises)

    return {
      payload: temperatures.map(t => t.payload)
    }
}

exports.readTemperatures = readTemperatures
```

Above implementation has a drawback, the device group won't be able to handle any other message until the querying the devices is completed. That mean we can handle a single query at a time. If one of the devices is slow due to a full mailbox it will cause our entire system to slow down.

In the next part of the tutorial we will explore an advanced implementation for querying the devices which solves the above issue.

Let's update the `template.yaml` under resources section with our new function:

```yaml
  ReadTemperatures:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: device-group-read-temperatures
      Handler: src/device-group.readTemperatures
      Runtime: nodejs8.10
      Policies:
        - AmazonDynamoDBFullAccess
        - AWSLambdaVPCAccessExecutionRole
```   

## Testing

Build the template and run the sam local:

```shell
sam build
sam local start-lambda --docker-network mqless-local --host 0.0.0.0 # Or npm start if you added it as your start script.
```

For the sake of the test, the address of our device group actor will be `g1` and our devices `g1d1` and `g1d2`.
First let's register some devices under our device group.

```shell
curl --data '{"deviceId":"g1d1"}' http://localhost:34543/request/device-group-register/g1
curl --data '{"deviceId":"g1d2"}' http://localhost:34543/request/device-group-register/g1
```
Let's record some temperatures:

```shell
curl --data '{"value":25}' http://localhost:34543/request/device-record-temperature/g1d1
curl --data '{"value":28}' http://localhost:34543/request/device-record-temperature/g1d2
```

And query the device group:

```shell
curl --data '{}' http://localhost:34543/request/device-group-read-temperatures/g1
```

## Summary

We created our device group, which register devices and read temperatures from all devices.
Register a device is simple, and because actors are always alive we don't need to create them on a cluster.

Our read temperatures implementation, however, has a drawback, it blocks the device group actor until all devices reply with the temperature.

In the next part we will use a better approach for querying the devices which doesn't block the device-group.
