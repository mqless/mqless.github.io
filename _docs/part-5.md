---
title: Part 5 - Deploy to AWS
permalink: /docs/part-5/
---

Create a new s3 bucket to host our package:

```shell
aws s3api create-bucket --bucket iot-mqless
```

You must use a unique name (globally) for a bucket, so just pick a different bucket name.

Packaging our application and upload to s3

```shell
sam package --s3-bucket iot-mqless --output-template-file packaged.yaml
```

Change the bucket name to your bucket name.

It can take a few minutes to upload the template to s3.

Finally, lets deploy our application:

```shell
sam deploy --template-file packaged.yaml --stack-name iot-mqless --capabilities CAPABILITY_IAM
```
