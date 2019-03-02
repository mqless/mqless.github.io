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

## Configure VPC Access

Because MQLess is installed within a VPC and the actors are connect to it, we need to allow the functions to access the VPC.

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

With the output of above create a Globals section and add it to template.yaml.

```yaml
Globals:
  Function:
    VpcConfig:
      SecurityGroupIds:
        - sg-your id
      SubnetIds:
        - subnet-1-your-id
        - subnet-2-your-id
```

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
