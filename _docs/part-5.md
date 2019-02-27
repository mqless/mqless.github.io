---
title: Part 5 - Deploy to AWS
permalink: /docs/part-5/
---

## Introduction

In the last part we are going to deploy our application to your AWS account.

## Prepare to package

Before we can package we need to instruct SAM to exclude `.aws-sam` directoy from our package.
SAM is using npm pack to package each of the Lambdas, to execlude the directory create a .npmignore file and add `.aws-sam` to it:

```shell
touch .npmignore
echo '.aws-sam' >> .npmignore
```

Create a new s3 bucket to host our package:

```shell
aws s3api create-bucket --bucket iot-mqless
```

You must use a unique name (globally) for a bucket, so just pick a different bucket name.

## Configure VPC Access

Because MQLess is installed within a VPC and the actors connect to it, we need to allow the functions to access the VPC and security group.

We need both the security group id of the MQLess machine and all the subnets of the VPC where MQLess reside.

Assuming you tagged MQLess with `mqless` tag run the following to get the security group id:

```shell
 aws ec2 describe-instances --filters "Name=tag-key,Values=mqless" --query 'Reservations[*].Instances[*].[SecurityGroups]' --output text | awk '{print $1}'
```

To get the subnets run the following:

```shell
MQLESS_VPC_ID=$(aws ec2 describe-instances --filters "Name=tag-key,Values=mqless" --query 'Reservations[*].Instances[*].[VpcId]' --output text | awk '{print $1}')
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$MQLESS_VPC_ID" | awk '{print $12}'
```

With the output of above create a globals section and add it to template.yaml.

```yaml
Globals:
  Function:
    Policies:
      - AWSLambdaVPCAccessExecutionRole
    VpcConfig:
      SecurityGroupIds:
        - sg-0f43ddfe5f2994432
      SubnetIds:
        - subnet-0b801a970112344b0
```

We also added the AWSLambdaVPCAccessExecutionRole policy which is needed when connecting to a VPC.

## Build and Package

Build and package our application:

```shell
sam build
sam package --s3-bucket iot-mqless --output-template-file packaged.yaml
```

Change the bucket name to your bucket name.

It can take a few minutes to upload the template to s3.

## Deploying the package

Finally, lets deploy our application:

```shell
sam deploy --template-file packaged.yaml --stack-name iot-mqless --capabilities CAPABILITY_IAM
```
