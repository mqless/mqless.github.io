---
title: Tutorial Part 2
permalink: /docs/tutorial-part-2/
---

## Introduction

In this part of the guide we will create the device group actor.
Device group is a collection of multiple devices, usually represent a home.
Device group actor main job is to query all of the devices in the group and return the list of temperature to the user.
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

    return {deviceAddress: `device-${deviceId}`}
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

Querying the device group is a little more complicated then register a device.
First in-order to query the device group we first need a way 
