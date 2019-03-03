---
title: Part 2 - Device
permalink: /docs/part-2/
---

## Introduction

With MQLess each actor is a Lambda function that handles messages sent to the actor.
We distinguish between the different messages using the message subject.

The device should read and record the temperature.
We will save the temperature in the actor state and read it from when requested.

## Device Actor

Create `device.js` under `src` directory.

```javascript
const {getState, putState} = require('./state')

async function readTemperature(message) {
    const {address} = message
    const state = await getState(address)

    if (!state || !state.lastTemperatureReading)
        return {subject: 'TemperatureNotAvailable'}

    return { subject: 'RespondTemperature', body: {temperature: state.lastTemperatureReading }}
}

async function recordTemperature(message) {
    const {address} = message
    const state = {lastTemperatureReading: message.body.temperature}

    await putState(address, state)

    return { subject: 'TemperatureRecorded'}
}

exports.handler = function (message) {
    console.log(message)

    switch (message.subject) {
        case 'ReadTemperature':
            return readTemperature(message)
        case 'RecordTemperature':
            return recordTemperature(message)
    }
}
```

As you can see, our device is pretty simple, we read the temperature from the state and write to the state.

## Testing

To test our device we need to add it to the SAM template.
Copy the following to the `template.yaml` file under `Resources` section:

```yaml
Device:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: Device
    Handler: src/device.handler
    Runtime: nodejs8.10
    Policies: AmazonDynamoDBFullAccess

```      

Build the SAM template with `sam build`.
To start SAM Lambda server run

```bash
sam local start-lambda --docker-network mqless-local --host 0.0.0.0
```

> Add `sam local start-lambda --docker-network mqless-local --host 0.0.0.0 --skip-pull-image` to package.json as your start script.

We are now ready to test the device actor. First, let's record a temperature, we will use 'A' as our actor address:

```shell
curl --data '{"temperature": "25"}' http://localhost:34543/send/Device/A/RecordTemperature
```

Read the temperature with:
```shell
curl --data '{}' http://localhost:34543/send/Device/A/ReadTemperature
```

If you will try a different address we will get the `TemperatureNotAvailable` message.
```shell
curl --data '{}' http://localhost:34543/send/Device/B/ReadTemperature
```

## Summary

We created the device actor which record and read temperatures.
We also created a SAM template and run the actor locally.

On the next part of the tutorial we will create the device group actor and see how it manage devices.
