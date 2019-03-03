---
title: Part 5 - Deploy to AWS
permalink: /docs/part-5/
---

## Introduction

In the last part, we are going to deploy our application to your AWS account.

## Prepare to package

Before we can package we need to instruct SAM to exclude `.aws-sam` directory from our package.
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
