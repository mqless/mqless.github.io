---
title: Part 2 - Device
permalink: /docs/part-2/
---


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
