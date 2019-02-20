---
title: Install on AWS EC2
permalink: /docs/install-ec2/
---

## Creating an IAM Role
You must create an IAM role for MQLess before you can use MQLess.

### To create an IAM role using the IAM console
1. Open the IAM console at https://console.aws.amazon.com/iam/
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
1. Create a new EC2 instance, and pick Ubuntu Server 18.041
2. Choose the size of the instance, start small and increase it with your demand. MQLess saves all pending messages in memory, so pick a machine with enough memory.
3. On the next page, choose the IAM role you created earlier.
4. Add storage, MQLess is not using the storage (yet), so it will only be for the operating system and page file.
5. Add any tags you need and next create the security group for MQLess. MQLess is using port 34543 by default, so open that port for other security groups that will be allowed to send messages to the actors. Also open My IP for testing purposes. 
6. Click Review and Done and create the machine.

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
git clone https://github.com/somdoron/mqless.git
cd mqless
./autogen.sh
./configure --with-systemd-units
make
sudo make install
sudo ldconfig
sudo systemctl daemon-reload
```

To run MQLess from console run mqless
To install and run MQLess as a service run:
```shell
sudo systemctl enable mqless
sudo systemctl start mqless
```

## Testing
In order to test MQLess we need to create a special lambda for the testing purposes, this should be a simple lambda that returns Hello World, You can also just return the request. Name the lambda mqless-testing-function or anything else with meaningful name. To test it run the following:

```shell
curl --data '{"msg":"Hello"}' http://MQLESS_HOST:34543/send/ROUTING_KEY/mqless-testing-function
```

Change MQLESS_HOST with the IP address of MQLess. You can also change the message or routing key, but for testing it is less important.
That it, you now installed MQLess on an EC2 machine.
