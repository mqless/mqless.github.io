---
title: Part 3 - Device Group
permalink: /docs/part-3/
---

## Introduction

In this part of the guide, we will create the device group actor.
A Device group is a collection of multiple devices, that usually represent a home.
Device group actor main job is to query all of the devices in the group and return the list of temperatures to the user.
In order to do that the actor first needs to know which devices are part of the group, so a device first needs to register with the device group.

## Register

The registration protocol using MQLess is very simple because actors in MQLess are always alive. We don't really need to create the device actor, we only need to add the new device id to the list of devices managed by the group.

Create `device-group.js` under `src` directory.

```javascript
const {getState, putState} = require('./state')
const uuid = require('uuid/v4')

async function requestTrackDevice(message) {
    const {address} = message
    const {deviceId} = message.body

    let state = await getState(address)
    if (!state)
        state = {devices: []}

    state.devices.push(deviceId)
    await putState(address, state)

    return {
        subject: 'DeviceRegistered',
        body: {
            deviceAddress: `Device/${deviceId}`
        }
    }
}

exports.handler = function (message) {
    console.log(message)

    switch (message.subject) {
        case 'RequestTrackDevice':
            return requestTrackDevice(message)
    }
}
```

When `RequestTrackDevice` is sent the actor adds the device id to the list of the devices managed by the actor.
However, more is happening here and we can begin to see the benefit of the actor model.
What would happen if two devices would register at the exact same time without MQLess? Chances are, only one device will get registered.
However, with MQLess and actor model, the problem is solved, because the register messages will be handled one by one and both devices will be added to the list of devices.

## List Devices

Our actor also need to provide the user all the devices registered with the actor, so let's add a function that will return the list of devices.
Add the following to `src/device-group.js`:

```javascript
async function requestDeviceList(message) {
    const {address} = message
    const state = await getState(address)
    const {devices} = state

    return {
        subject: 'ReplyDeviceList',
        body: {
            devices
        }
    }
}
```

Also add the handling in the handle function:

```javascript
exports.handler = function (message) {
    console.log(message)

    switch (message.subject) {
        case 'RequestTrackDevice':
            return requestTrackDevice(message)
        case 'RequestDeviceList':
            return requestDeviceList(message)
    }
}
```

## Testing

First, let's add our actor to SAM template file:

```yaml
DeviceGroup:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: DeviceGroup
    Handler: src/device-group.handler
    Runtime: nodejs8.10
    Policies: AmazonDynamoDBFullAccess
```

Build the template and run the SAM local:

```shell
sam build
sam local start-lambda --docker-network mqless-local --host 0.0.0.0 # Or npm start if you added it as your start script.
```

For the sake of the test, the address of our device group actor will be `G1` and our devices `G1D1` and `G1D2`.
First let's register some devices under our device group.

```shell
curl --data '{"deviceId":"G1D1"}' http://localhost:34543/send/DeviceGroup/G1/RequestTrackDevice
curl --data '{"deviceId":"G1D2"}' http://localhost:34543/send/DeviceGroup/G1/RequestTrackDevice
```

Request list of devices:

```shell
curl --data '{}' http://localhost:34543/send/DeviceGroup/G1/RequestDeviceList
```

## Summary

We created our device group, which register devices and list devices.
Register a device is simple, and because actors are always alive we don't need to create them on a cluster.

In the next part, we will request all temperatures from the device group.
