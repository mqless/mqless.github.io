---
title: Part 4 - Device Group Query
permalink: /docs/part-4/
---

## Introduction

Our device group also need to return the temperatures from all devices.
So we first need to learn how an actor can send messages to other actors and handle replies.

But first, let's talk about a little problem, requesting temperatures from all devices is an asynchronous process.
We need to message each of the devices in the group, collect all responses and then send the reply back to the user.

We can implement all of this within the device-group, saving the replies we are waiting for and replies so far in the state and eventually send the response back.
If we implement it in the device-group we would only support one query at a moment. Or need to have a more complex code to support multiple queries.

However, we can use a better pattern for this. We can have an actor that will represent the query process and keep all the state.

## Send a message to another actor

So, we first need to understand how we can send messages to other actors. We will use that to send `ReadTemperature` message to the device actors.
To send a message to another actor, include a `send` field as part of the json returned by the actor.
The send can either be an array or a single object. It must have the `to` and `subject` fields and may include the `body` field.
For example:

```javascript
  return {
    send: {
      to: 'Device/A',
      subject: 'ReadTemperature'
    }
  }
```

## Forward a Message

Additional to send, an actor can also forward a message.
Forwarding is very similar to send, the difference is that forward keep the original sender of the message.
Allow the other actor to reply directly to the sender without going through the forwarding actor.
Also, forward must be a single object and cannot be an array, for example:

```javascript
return {
  forward: {
    to: 'DeviceGroupQuery/B',
    subject: 'RequestAllTemperatures',
    body: {
      devices: ['A']
    }
  }
}
```

## Asynchronous Reply

Until now, all of our actors send a reply back immediately.
However, with our asynchronous process, we don't have the reply ready yet, so both need a way to delay the reply and send it back later.

Actually, replying a request directly is just a syntactic sugar to send a message, so:

```
return {
  subject: 'Hello'
}
```

Is equivalent to:

```
return {
  send: {
    to: message.from,
    subject: 'Hello'
  }
}
```

So, to delay the reply we need to store the original from field and send the reply later.

## Device Group Revisited

Let's go back to the device group. We need to implement the `RequestAllTemperatures` and forward it to the query device.

```javascript
async function requestAllTemperatures(message) {
    const {address} = message
    const state = await getState(address)
    const {devices} = state

    // We generate random address for the new device group query
    const queryId = uuid()

    return {
        forward: {
            to: `DeviceGroupQuery/${queryId}`,
            subject: 'RequestAllTemperatures',
            body: {devices}
        }
    }
}
```

The `requestAllTemperatures` function forward the message to the query device with the list of devices.

As you can notice, we are using a new package to generate a unique address for the query device.
Install the `uuid` package by running `npm install --save uuid` and also import the function at the head of the file with:
```javascript
const uuid = require('uuid/v4')
```

Add the handling to the handler function:

```javascript
exports.handler = function (message) {
    console.log(message)

    switch (message.subject) {
        case 'RequestTrackDevice':
            return requestTrackDevice(message)
        case 'RequestDeviceList':
            return requestDeviceList(message)
        case 'RequestAllTemperatures':
            return requestAllTemperatures(message)
    }
}
```

## Query Actor

The query actor is going to do the following:
1. Send ReadTemperature message to all of the devices
2. Wait for RespondTemperature (or TemperatureNotAvailable) messages from all devices
3. Send the reply back using a send

In order to achieve that the query actor will store the original from and the list of devices in the actor's state.

```javascript
const {getState, putState} = require('./state')

async function requestAllTemperatures (message) {
    const {address, from} = message
    const {devices} = message.body

    const state = {
        stillWaiting: devices.map(deviceId => `Device/${deviceId}`),
        repliesSoFar: [],
        from: from
    }
    await putState(address, state)

    // Send a ReadTemperature message to each of the actors concurrently and collecting the responses.
    const send = devices.map(deviceId => {
        return {
            to: `Device/${deviceId}`,
            subject: 'ReadTemperature'
        }
    })

    return {send}
}

async function onRespondTemperature(message) {
    const {address, from} = message
    const {temperature} = message.body

    const state = await getState(address)

    state.repliesSoFar.push(temperature)
    state.stillWaiting = state.stillWaiting.filter(deviceAddress => deviceAddress !== from)
    await putState(address, state)

    if (state.stillWaiting.length === 0) {
        return {
            send: {
                to: state.from,
                subject: 'RespondAllTemperatures',
                body: {
                    temperatures: state.repliesSoFar
                }
            }
        }
    }

    return {}
}

async function onTemperatureNotAvailable(message) {
    const {address, from} = message

    const state = await getState(address)

    state.stillWaiting = state.stillWaiting.filter(deviceAddress => deviceAddress !== from)
    await putState(address, state)

    if (state.stillWaiting.length === 0) {
        return {
            send: {
                to: state.from,
                subject: 'RespondAllTemperatures',
                body: {
                    temperatures: state.repliesSoFar
                }
            }
        }
    }

    return {}
}

exports.handler = function (message) {
    console.log(message)

    switch (message.subject) {
        case 'RequestAllTemperatures':
            return requestAllTemperatures(message)
        case 'RespondTemperature':
            return onRespondTemperature(message)
        case 'TemperatureNotAvailable':
            return onTemperatureNotAvailable(message)
    }
}
```

Add the new actor to the template file under the resources section:

```yaml
DeviceGroupQuery:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: DeviceGroupQuery
    Handler: src/device-group-query.handler
    Runtime: nodejs8.10
    Policies: AmazonDynamoDBFullAccess
```

## Testing

Let's record some temperatures:

```shell
curl --data '{"temperature":25}' http://localhost:34543/send/Device/G2D1/RecordTemperature
curl --data '{"temperature":28}' http://localhost:34543/send/Device/G2D2/RecordTemperature
```

Let's track them in the group:

```shell
curl --data '{"deviceId":"G2D1"}' http://localhost:34543/send/DeviceGroup/G2/RequestTrackDevice
curl --data '{"deviceId":"G2D2"}' http://localhost:34543/send/DeviceGroup/G2/RequestTrackDevice
```

And finally request all temperatures:

```shell
curl --data '{}' http://localhost:34543/send/DeviceGroup/G2/RequestAllTemperatures
```

## Summary

We learned how actors can send messages to other actors and the asynchronous process pattern.
