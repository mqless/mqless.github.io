---
title: Install on AWS EC2
permalink: /docs/install-ec2/
---

## Create VPC Endpoint for DynamoDB
DynamoDB is needed in order to store the state of the actors.
In order for our Lambda to be able to access DynamoDB, we need to create a VPC Endpoint for DynamoDB.

1. Open the [VPC console](https://console.aws.amazon.com/vpc)
2. In the navigation pane click Endpoints
3. Click Create Endpoint
4. Check 'AWS Services'
5. Search for the DynamoDB service name, for example `com.amazonaws.us-east-1.dynamodb`
6. Pick the VPC, this is the same VPC you will use for MQLess and Lambda
7. Click Create Endpoint
8. Go to the newly created endpoint, click the Route Tables tab and click 'Manage Route Tables'
9. Check the route table you will be using for Lambda.

## Creating an IAM Role
You must create an IAM role for MQLess before you can use MQLess.

### To create an IAM role using the IAM console
1. Open the [IAM console](https://console.aws.amazon.com/iam/)
2. In the navigation pane, choose Policies, Create Policy.
3. Choose the Lambda service
4. Expand the Write section and check InvokeFunction
5. Expand Resources, check Specific and check Any
6. Click Review Policy, name it (like InvokeLambda) and click Create Policy.
7. In the navigation pane, choose Roles, Create role.
8. Choose the EC2 service and click Next: Permissions
9. Check the policy we just created and click Next: Tags
10. Click Next: Review, name the new role (like MQLessRole), give it a description and click Create Role

## Creating the EC2 Instance
1. Create a new [EC2 instance](https://console.aws.amazon.com/ec2/), and pick Ubuntu Server 18.041
2. Choose the size of the instance, start small and increase it with your demand. MQLess saves all pending messages in memory, so pick a machine with enough memory and click next.
3. MQLess should be used within a VPC. Choose a VPC network and subnet.
4. Choose the IAM role you created earlier.
5. Add storage, MQLess is not using the storage (yet), so it will only be for the operating system and page file.
6. Add tag `mqless` without a value and click next.
7. Create a security group for MQLess. MQLess is using port 34543 by default, so open that port for other security groups that will be allowed to send messages to the actors. Also, consider opening your own IP for testing purposes.
8. Click Review and Done and create the machine.

## Installing MQLess
Connect to the machine through ssh and run the following:
```shell
sudo apt update

sudo apt install -y libtool automake autoconf pkg-config build-essential libnsspem libcurl4-nss-dev libmicrohttpd-dev libzmq3-dev libjansson-dev
git clone https://github.com/zeromq/czmq.git
cd czmq
./autogen.sh
./configure
make
sudo make install
cd ..
#install mqless
git clone https://github.com/mqless/mqless
cd mqless
./autogen.sh
./configure --with-systemd-units
make
sudo make install
sudo ldconfig
sudo systemctl daemon-reload
```

To run MQLess from console run `mqless`

To install and run MQLess as a service run:
```shell
sudo systemctl enable mqless
sudo systemctl start mqless
```

## Testing
In order to test MQLess, we need to create a special [AWS Llambda](https://console.aws.amazon.com/ec2/) for the testing purposes. This should be a simple lambda that returns Hello World. You can also just return the request. Name the lambda mqless-testing-function or anything else with a meaningful name. To test it run the following:

```shell
curl --data '{"msg":"Hello"}' http://MQLESS_HOST:34543/request/mqless-testing-function/some-address
```

Change:
* MQLESS_HOST - the IP address of mqless
* 34543 - this is the default port used by mqless
* mqless-testing-function - the name of the Lambda you created

You can also change the message, the address, but for testing it is less important.
That is all, you now have installed MQLess on an EC2 machine.
