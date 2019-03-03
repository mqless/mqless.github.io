---
title: Part 2 - Device
permalink: /docs/part-2/
---

## Introduction

The device should read and record the temperature.
We will save the temperature in the actor state and read it from when requested.

## Device Actor

Create `device.js` under `src` directory.

```javascript
const {getState, putState} = require('./state')

async function readTemperature(message) {
    const {address} = message.header
    const state = await getState(address)

    if (!state)
        return {value: null}

    return {payload: state.lastTemperatureReading}
}

async function recordTemperature(message) {
    const {address} = message.header
    const state = {lastTemperatureReading: message.payload.value}

    await putState(address, state)

    return {payload: {}}
}

exports.readTemperature = readTemperature
exports.recordTemperature = recordTemperature
```

As you can see, our device is pretty simple, we read the temperature from the state and write to the state.

> It is a good practice to send the response wrap in a payload field, as MQLess support other types of return values from the Lambda (e.g forward) and may add additional information to the returned json.

We also need to update our SAM template. Add the following to the `template.yaml` file under `Resources` section:

```yaml
  ReadTemperature:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: device-read-temperature
      Handler: src/device.readTemperature
      Runtime: nodejs8.10
      Policies:
        - AmazonDynamoDBFullAccess
        - AWSLambdaVPCAccessExecutionRole

  RecordTemperature:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: device-record-temperature        
      Handler: src/device.recordTemperature
      Runtime: nodejs8.10
      Policies:
        - AmazonDynamoDBFullAccess
        - AWSLambdaVPCAccessExecutionRole
```      

## Testing

We are almost ready to run our actor, but first, we need to build the SAM template with `sam build`.
To start SAM Lambda server run `sam local start-lambda --docker-network mqless-local --host 0.0.0.0`.

> Add `sam local start-lambda --docker-network mqless-local --host 0.0.0.0` to package.json as your start script.

We are now ready to test the device actor. First, let's record a temperature, we will use 'A' as our actor address:

```shell
curl --data '{"value": "25"}' http://localhost:34543/request/device-record-temperature/A
```

Read the temperature with:
```shell
curl --data '{}' http://localhost:34543/request/device-read-temperature/A
```

If you will try a different address we will get an empty result:
```shell
curl --data '{}' http://localhost:34543/request/device-read-temperature/A
```

## One Lambda per Actor

In the above example, for each function of the device (read and record), we created a different Lambda.
However, that is not always the best approach. Instead, we can have one function for each actor, with the subject field distinguishing between the message types.

The benefit for this will come later when we would want to cache the state between actor calls, batch messages or change behavior.
MQLess is agnostic to which type you use, for the tutorial, we will continue to use Lambda per each message that the actor support.

## Summary

We created the device actor which record and read temperatures.
We also created a SAM template and run the actor locally.

On the next part of the tutorial we will create the device group actor and see how it manage and query devices.
