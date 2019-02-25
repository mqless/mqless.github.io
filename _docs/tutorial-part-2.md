---
title: Tutorial Part 2
permalink: /docs/tutorial-part-2/
---

## Introduction

In this part of the guide we will create the device group actor.
Device group is a collection of multiple devices, usually represent a home.
Device group actor main job is to query all of the devices in the group and return the list of temperatures to the user.
In order to do that the actor first need to know which devices are part of the group, so device first need to register with the device group.

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

    return {deviceAddress: deviceId}
}

exports.registerDevice = registerDevice
```

So when the register function is being called we just add the device id to the list of the devices managed by the actor.
However, more is happening here and we can see the benefit of the actor model.
What would happen if two devices would register at the exact same time without MQLess? Chances are, only one device will get registered.
However, with MQLess and actor model, the problem is solved, because the register messages will be handled one by one and both devices will be added to the list of devices.

We need to update our SAM template as well, add the following to `template.yaml` under the resources section:

```yaml
  RegisterDevice:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/device-group.registerDevice
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess
```

## Querying the device group

For querying devices we would need the axios package in order to make http requests, run `npm install --save axios`.

Querying the device group is a little more complicated than register a device.
In-order to query the devices in the group we need to send them all the read temperature message.
However, how can actor send messages to other actors? This is actually at the base of actor model.

The answer is, the same way we tested our actors, through MQLess API. Now the problem is, how the actor discover the endpoint of MQLess? Hard coded? configuration? Luckily, MQLess provide the endpoint in the header of the message, so you don't need any configuration and it works both locally and on AWS.
 
We will start with a naive implementation for querying the device. Sending the read temperature message to each of the devices and wait for responses from all of them. 

```javascript
const {post} = require('axios')

async function queryDevices(message) {
    const {address, endpoint} = message.header
    let state = await getState(address)

    if (!state)
        return {temperatures: []}

    // Send a ReadTemperature message to each of the actors concurrently and collecting the responses.
    const promises = state.devices.map(deviceId =>
        post(`${endpoint}/send/ReadTemperature/${deviceId}`).then(response => response.data))

    const temperatures = await Promise.all(promises)

    return {temperatures}
}

exports.queryDevices = queryDevices
```

Above implementation have a drawback, the device groups won't be able to handle any other message until the querying the devices is completed. That mean we can handle a single query at a time. If one of the devices is slow due to full mailbox it will cause our entire system to slow down.

In the next part of the tutorial we will explore an advanced implementation for querying the devices which solves the above issue.

Let's update the `template.yaml` under resources section with our new function:

```yaml
  QueryDevices:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/device-group.queryDevices
      Runtime: nodejs8.10
      Policies: AmazonDynamoDBFullAccess
```   

## Testing

For the sake of the test, the addres of our device group actor will be `g1` and for our devices `g1d1` and `g1d2`.
First lets create some devices under our device group.

```shell
curl --data '{"deviceId":"g1d1"}' http://localhost:34543/send/RegisterDevice/g1
curl --data '{"deviceId":"g1d2"}' http://localhost:34543/send/RegisterDevice/g1
```

We can now query our device group with:

```shell
curl --data '{}' http://localhost:34543/send/QueryDevices/g1
```

We didn't record any temperatures so the results are nulls.
Let's record some temperatures:

```shell
curl --data '{"value":25}' http://localhost:34543/send/RecordTemperature/g1d1
curl --data '{"value":28}' http://localhost:34543/send/RecordTemperature/g1d2
```

And query the device group again

```shell
curl --data '{}' http://localhost:34543/send/QueryDevices/g1
```

## Summary

We created our device group, which register devices and query all devices.
Register a device is simple, and because actors are always alive we don't need to create them on the cluster.

Our querying devices implementation, however, has a drawback, it blocks the device group actor until all devices reply with the temperature. 

In the next part we will use a better approch for querying the devices which doesn't block the device-group.





