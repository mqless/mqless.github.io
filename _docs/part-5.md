---
title: Part 5 - Deploy to AWS
permalink: /docs/part-5/
---

## Introduction

In the last part, we are going to deploy our application to your AWS account.

## Prepare to package

Before we can package, we need to instruct SAM to exclude `.aws-sam` directory from our package.
SAM is using npm pack to package each of the Lambdas. To exclude the directory create a .npmignore file and add `.aws-sam` to it:

```shell
touch .npmignore
echo '.aws-sam' >> .npmignore
```

Create a new s3 bucket to host our package:

```shell
aws s3api create-bucket --bucket iot-mqless
```

You must use a unique name (globally) for a bucket, so pick a different bucket name for your bucket.

## Build and Package

Build and package our application:

```shell
sam build
sam package --s3-bucket iot-mqless --output-template-file packaged.yaml
```

Change the bucket name to your bucket name.

It can take a few minutes to upload the template to s3.

## Deploying the package

Finally, let's deploy our application:

```shell
sam deploy --template-file packaged.yaml --stack-name iot-mqless --capabilities CAPABILITY_IAM
```

## Testing

Let's first define an environment variable with endpoint of the remote MQLess you installed on AWS EC2 machine:

```shell
MQLESS_ENDPOINT = http://YOUR_MQLESS_ADDRESS:34543
echo $MQLESS_ENDPOINT
```

Let's record some temperatures:

```shell
curl --data '{"temperature":25}' $MQLESS_ENDPOINT/send/Device/G2D1/RecordTemperature
curl --data '{"temperature":28}' $MQLESS_ENDPOINT/send/Device/G2D2/RecordTemperature
```

Let's track them in the group:

```shell
curl --data '{"deviceId":"G2D1"}' $MQLESS_ENDPOINT/send/DeviceGroup/G2/RequestTrackDevice
curl --data '{"deviceId":"G2D2"}' $MQLESS_ENDPOINT/send/DeviceGroup/G2/RequestTrackDevice
```

And finally request all temperatures:

```shell
curl --data '{}' $MQLESS_ENDPOINT/send/DeviceGroup/G2/RequestAllTemperatures
```
