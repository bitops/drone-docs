---
date: 2000-01-01T00:00:00+00:00
title: Amazon
author: eoinmcafee
weight: 100
---

<div class="alert">
The Amazon driver is in the Release Candidate phase.
</div>

# Overview

**By default it will create a pool with a max size of 2 running Ubuntu 18.04. The pools is called `testpool`.**

Amazon specific configuration in a pool file.

## Authentication

By default we require access_key_id and access_key_secret which is needed for create an the instance.

__Alternatively__ use an IAM role to manage pool instances on aws drone runner. To use the IAM role, aws runner needs to run on EC2 instance with IAM role having CRUD permissions on EC2. This will allow the runner to use the instance’s IAM role to get temporary security credentials to make calls to AWS for managing pool & removes requirement of specifying `access_key_id` and `access_key_secret`.

## VPC

By default it will use the default VPC for that user or you can specify the VPC id.

## Security groups

By default it will create the necessary security group. It is named "harness runner".

__Alternatively__ you can specify your own security group and passing its ID to the pool file. Firewall rules for the build instances [ec2 authorizing-access-to-an-instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html) We need allow ingress and egress access to port 9079. Once complete you will have a security group id, which is needed for configuration of the runner.

- (optional) For debugging purposes, you can amend the security group with the following rules:

  - `SSH TCP 22   0.0.0.0/0` for linux.
  - `RDP TCP 3389 0.0.0.0/0` for windows.

  This will allow you to remotely connect to the build instances. Once you set `key_pair_name`.

# Pool Spec

Cloud specific configuration.

{{< highlight yaml "linenos=table" >}}
  account          Account   # explained in section below
  ami              string    # ami id
  disk             Disk      # explained in section below
  hibernate        bool      # hibernate instance when added to the pool (amazon-linux only)
  iam_profile_arn  string    # arn of the iam profile to apply to the instance
  network          Network   # explained in section below
  size             string    # t2.nano, m4.large, etc
  tags             []string  # tags to apply to the instance
  user_data        string    # user data to apply to the instance
  user_data_path   string    # path to user data script
  vpc              string    # vpc id
{{< / highlight >}}

More information on user_data and user_data_path can be found [custom cloud-init]({{< relref "../../configuration/cloud-init.md" >}})

## Account

Contains the AWS account configuration.

{{< highlight yaml "linenos=table" >}}
  access_key_id     string   # access key id
  access_key_secret string   # access key secret
  region            string   # aws region
{{< / highlight >}}

## Disk

Contains AWS block information:

{{< highlight yaml "linenos=table" >}}
  size int      # size in GB
  type string   # gp2, io1, standard
  iops string   # iops for io1
{{< / highlight >}}

## Network

Contains AWS network information:

{{< highlight yaml "linenos=table" >}}
  vpc_security_groups []string  # vpc security groups
  security_groups     []string  # security group ids, default it will use the security for 'harness runner'
  subnet_id           string    # subnet id
  private_ip          bool      # assign private ip
{{< / highlight >}}

# Recommended AMIs

## [Ubuntu 20.04](https://aws.amazon.com/marketplace/pp/prodview-iftkyuwv2sjxi?sr=0-2&ref_=beagle&applicationId=AWSMPContessa)

This is the default AMI for the runner.

## [Windows Server 2019 with containers](https://aws.amazon.com/marketplace/pp/prodview-iehgssex6veoi)

NB: be sure to set the platform to windows

  ```yaml
version: "1"
instances:
- name: ubuntu-aws
  default: true
  type: amazon
  platform:
    os: windows
```

NB Docker support in windows server 2019 does not use the same docker engine as Windows 10/11 (with WSL2/HyperV). It does not support all of the features of modern Docker on Windows, eg passing through virtualisation directly to the container. There is some more information from AWS [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_Windows.html).

## [Amazon Linux 2](https://aws.amazon.com/marketplace/pp/prodview-zc4x2k7vt6rpu?sr=0-1&ref_=beagle&applicationId=AWSMPContessa)

NB: be sure to set the platform to linux, and set os_name to amazon-linux to use this AMI. **Hibernate is supported.**

```yaml
version: "1"
instances:
- name: ubuntu-aws
  default: true
  type: amazon
  platform:
    os: linux
    os_name: amazon-linux
  spec:
    account:
        region: us-east-2
        availability_zone: us-east-2c
        access_key_id: XXXXXXXXXXXXXXXXXXXXX
        access_key_secret: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    hibernate: true
```

Depending on the AMI's you are using, you may need to subscribe to it. We have tested against Ubuntu 20.04 and Windows 2019 with containers.

# Example pool setup

EG, This `pool.yml` file configures 2 pools each with a pool size of 2 and a limit of 4.

{{< highlight yaml "linenos=table" >}}
version: "1"
instances:
  - name: ubuntu-aws
    default: true
    type: amazon
    pool: 1    # total number of warm instances in the pool at all times
    limit: 4   # limit the total number of running servers. If exceeded block or error.
    platform:
      os: linux
      arch: amd64
    spec:
      account:
        region: us-east-2
        availability_zone: us-east-2c
        access_key_id: XXXXXXXXXXXXXXXXXXXXX
        access_key_secret: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
      ami: ami-051197ce9cbb023ea
      size: t2.nano
      network:
        security_groups:
          - XXXXXXXXXXXXXXXX
{{< / highlight >}}
