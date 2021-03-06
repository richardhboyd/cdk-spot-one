
[![NPM version](https://badge.fury.io/js/cdk-spot-one.svg)](https://badge.fury.io/js/cdk-spot-one)
[![PyPI version](https://badge.fury.io/py/cdk-spot-one.svg)](https://badge.fury.io/py/cdk-spot-one)
![Release](https://github.com/pahud/cdk-spot-one/workflows/Release/badge.svg)

# cdk-spot-one

One spot instance with EIP and defined duration. No interruption.

# Why

Sometimes we need an Amazon EC2 instance with static fixed IP for testing or development purpose for a duration of
time(probably hours). We need to make sure during this time, no interruption will occur and we don't want to pay
for on-demand rate. `cdk-spot-one` helps you reserve one single spot instance with pre-allocated or new
Elastic IP addresses(EIP) with defined `blockDuration`, during which time the spot instance will be secured with no spot interruption.

Behind the scene, `cdk-spot-one` provisions a spot fleet with capacity of single instance for you and it associates the EIP with this instance. The spot fleet is reserved as spot block with `blockDuration` from one hour up to six hours to ensure the high availability for your spot instance.

Multiple spot instances are possible by simply specifying the `targetCapacity` construct property, but we only associate the EIP with the first spot instance at this moment.

Enjoy your highly durable one spot instance with AWS CDK!

# Sample

```ts
import { SpotFleet } from 'cdk-spot-one';

// create the first fleet for one hour and associate with our existing EIP
const fleet = new SpotFleet(stack, 'SpotFleet')

// configure the expiration after 1 hour
fleet.expireAfter(Duration.hours(1))


// create the 2nd fleet with single Gravition 2 instance for 6 hours and associate with new EIP
const fleet2 = new SpotFleet(stack, 'SpotFleet2', {
  blockDuration: BlockDuration.SIX_HOURS,
  eipAllocationId: 'eipalloc-0d1bc6d85895a5410',
  defaultInstanceType: new InstanceType('c6g.large'),
  vpc: fleet.vpc,
})
// configure the expiration after 6 hours
fleet2.expireAfter(Duration.hours(6))

// print the instanceId from each spot fleet
new CfnOutput(stack, 'SpotFleetInstanceId', { value: fleet.instanceId })
new CfnOutput(stack, 'SpotFleet2InstanceId', { value: fleet2.instanceId })

```

# ARM64 and Graviton 2 support

`cdk-spot-one` selects the latest Amazon Linux 2 AMI for your `ARM64` instances. Simply select the instance types with the `defaultInstanceType` property and the `SpotFleet` will auto configure correct AMI for the instance.


```ts
defaultInstanceType: new InstanceType('c6g.large')
```


# SSH connect

By default the `cdk-spot-one` does not assign any SSH public key for you on the instance. You are encouraged to use `ec2-instance-connect` to send your public key from local followed by one-time SSH connect.

For example:

```sh
pubkey="$HOME/.ssh/aws_2020_id_rsa.pub"
echo "sending public key to ${instanceId}"
aws ec2-instance-connect send-ssh-public-key --instance-id ${instanceId} --instance-os-user ec2-user \
--ssh-public-key file://${pubkey} --availability-zone ${az} 
```

You may also create a simple `ec2-connect.sh` script like this and save in your $PATH:

```sh
#!/bin/bash

instanceId=$1
pubkey="$HOME/.ssh/aws_2020_id_rsa.pub"
sshUser='ec2-user'

az=$(aws ec2 describe-instances --instance-id ${instanceId} --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' --output text)
publicIp=$(aws ec2 describe-instances --instance-id ${instanceId} --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)


echo "sending public key to ${instanceId}"
aws ec2-instance-connect send-ssh-public-key --instance-id ${instanceId} --instance-os-user ${sshUser} \
--ssh-public-key file://${pubkey} --availability-zone ${az} > /dev/null

if [[ $2 != '--send-key-only' ]]; then
  echo "connecting to ${publicIp} at ${az}"
	ssh ${sshUser}@${publicIp}
fi
```

And simply run this to connect to the EC2 instance.

```sh
$ ec2-connect.sh i-01f827ab9de7b93a9
```

It's also possible to explicitly specify your existing SSH key with the `keyName` construct property if you like.
