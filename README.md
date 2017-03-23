# continuous-everything

## Overview

My attempt to automate as much of my hacking activity as possible - tests, deployment, etc.

For anything I need to do more than once, once I commit and push, everything else should happen automatically. 

I will attempt to list everything I had to do manually here, even if it's not in complete detail.
 
## Buildkite

The starting point for automation of test/build/delivery pipelines.

### Buildkite Manual Steps

* Get an account on AWS
* Enable billing alerts:
  * https://console.aws.amazon.com/billing/home?#/preferences
  * Create alarm under Cloudwatch -> Alerts -> Billing
* Get an account on https://buildkite.com
* Create an Org
* Create an Agent, select "AWS" type, and follow their instructions to set up
  the [Elastic CI Stack for AWS](https://github.com/buildkite/elastic-ci-stack-for-aws)
  * Consider taking their suggestions to "Follow best practice by setting up a separate development AWS account and using
    role switching and consolidated billing".
  * For now, I just made a 'buildkite' IAM user and group with full
    Administrator access
* Create buckets for secrets and artifacts, specify in CloudFormation form
* Tweak CloudWatch rules:
  * scale down after 5 minutes instead of 30
  * scale up after 3 minutes instead of 1

## Rancher + Spotinst on AWS Initial Setup

* Set up Rancher on AWS: https://docs.rancher.com/os/running-rancheros/cloud/aws/

### AWS One-Time Setup For Rancher + Spotinst

* Create 'rancher' IAM user and group with full Administrator access, save access key in lastpass
* Set up AWS CLI locally: `brew install awscli`
* `aws configure`
* Create Keypair:
  * `aws ec2 create-key-pair --region $REGION --key-name rancher` - save in lastpass
  * Copy private key locally to `~/.ssh/rancher.pem` with permissions 600
* Create and tag VPC:
  * `aws ec2 create-vpc --cidr-block '10.0.0.0/16'`
  * `aws ec2 create-tags --resource $VPC_ID --tags Key=Name,Value=rancher`
* Find and Name Network ACL:
  * `aws ec2 describe-network-acls --filters Name=vpc-id,Values=$VPC_ID`
  * `aws ec2 create-tags --resource $ACL_ID --tags Key=Name,Value=rancher`
* Create and Name Security Group:
  * `aws ec2 create-security-group --group-name rancher --description rancher --vpc-id=vpc-567e3831`
  * `aws ec2 create-tags --resource $SG_ID --tags Key=Name,Value=rancher`
* Create, Attach, and Name Internet Gateway:
  * `aws ec2 create-internet-gateway`
  * `aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID`
  * `aws ec2 create-tags --resource $IGW_ID --tags Key=Name,Value=rancher
* Find and Name Route Table:
  * `aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC_ID`
  * `aws ec2 create-tags --resource $RTB_ID --tags Key=Name,Value=rancher`
* Add route for Internet Gateway
  * `aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID`
* Create, Associate, and Name Subnets:
  * For each $AVAILABILITY_ZONE (a,b,c) and $CIDR_OCTET (101,102,103) with var SUBNET_ID_N (where N = 1,2,3)
    * Note: Only SUBNET_ID_1 is used for rancher server, other subnets are for use by SpotInst to
      create workers in different AZs.
    * `aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block "10.0.$CIDR_OCTET.0/24" --availability-zone $REGION$AVAILABILITY_ZONE`
    * `aws ec2 associate-route-table --subnet-id $SUBNET_ID_N --route-table-id $RTB_ID`
    * `aws ec2 modify-subnet-attribute --map-public-ip-on-launch --subnet-id $SUBNET_ID_N`
    * `aws ec2 create-tags --resource $SUBNET_ID_N --tags Key=Name,Value=rancher-$AVAILABILITY_ZONE`
* Create and Name EBS Volume to hold persistent Rancher Mysql DB data
  * `aws ec2 create-volume --volume-type gp2 --size 2 --availability-zone $REGION$AVAILABILITY_ZONE`
  * `aws ec2 create-tags --resource $VOL_ID --tags Key=Name,Value=rancher`
  * Create a standard ubuntu instance to ssh to and mount/format the volume:
    * `sudo mkfs -t ext4 /dev/xvdm`
* Create AWS route53 Public Hosted Zone and Get Zone ID
  * https://console.aws.amazon.com/route53/home#hosted-zones: - name after purchased $HOSTED_ZONE_NAME
  * `aws route53 list-hosted-zones-by-name --dns-name $HOSTED_ZONE_NAME`
  * `aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol all --cidr 0.0.0.0/0`
  * `aws ec2 authorize-security-group-egress --group-id $SG_ID --protocol all --cidr 0.0.0.0/0`

### AWS steps to Recreate RancherOS server instance

* Set env vars from one-time setup:
  * `source ./set-env-vars`
* Create and Name RancherOS server instance:
  * RancherOS AMI for $REGION region: https://github.com/rancher/os/blob/master/README.md
  * `aws ec2 run-instances --region $REGION --image-id $AMI_ID --count 1 --instance-type t2.micro --subnet-id $SUBNET_ID_1 --security-group-ids $SG_ID --key-name rancher`
  * `aws ec2 create-tags --resource $I_ID --tags Key=Name,Value=rancher-server`
* Attach MYSQL EBS volume:
  * `aws ec2 attach-volume --volume-id $VOL_ID --instance-id $I_ID --device /dev/xvdm`
* Assign DNS A record to instance's public IP:
  * Get Public IP Address: `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=$VPC_ID`
  * `aws route53 change-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID --change-batch "{\"Changes\":[{\"Action\":\"UPSERT\",\"ResourceRecordSet\":{\"Name\":\"rancher.$HOSTED_ZONE_NAME\",\"Type\":\"A\",\"TTL\":300,\"ResourceRecords\":[{\"Value\":\"$PUBLIC_IP_ADDRESS\"}]}}]}"`

### Rancher Server Setup

* SSH to rancherOS container
  * `ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/rancher.pem rancher@rancher.$HOSTED_ZONE_NAME` 
* Run rancher server docker image:
  * `sudo mkdir /var/lib/mysql`
  * `sudo mount /dev/xvdm /var/lib/mysql`
  * `sudo docker run -d -v /var/lib/mysql:/var/lib/mysql --name rancher-server --restart=unless-stopped -p 8080:8080 rancher/server`
* Initial Config
  * Log in via initial unauthenticated access: http://rancher.$HOSTED_ZONE_NAME:8080/
  * Set up github access control under "ADMIN" menu, follow instructions

### Spotinst one-time setup
* https://console.spotinst.com/#/wizard/aws
* follow instructions - put in appropriate values
* On "compute" tab, enter following user data cloud-config for rancherOS host instances - replace host and token:

```
#cloud-config
write_files:
  - path: /etc/rc.local
    permissions: "0755"
    owner: root
    content: |
      #!/bin/bash
      for i in {1..20}
      do
      docker info && break
      sleep 1
      done
      #Starting the Rancher Agent
      # Setting a CATTLE_HOST_LABELS of "spotinst.instanceId" which is REQUIRED for the Spotinst integration to work.
      sudo docker run -d -e CATTLE_HOST_LABELS="spotinst.instanceId=`wget -qO- http://169.254.169.254/latest/meta-data/instance-id`" --privileged -v /var/run/docker.sock:/var/run/docker.sock rancher/agent:v0.8.2 http://rancher.$HOSTED_ZONE_NAME:8080/v1/scripts/$TOKEN_FROM_RANCHER_START_HOST_SCREEN
```

## BuildKite Example Continuous Delivery Pipeline Example

### Manually create example docker image

* Docker notes
  * https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#workdir
  * http://www.carlboettiger.info/2014/08/29/docker-notes.html
* Create repo and Dockerfile
  * https://github.com/thewoolleyman/docker-webserver-example
* Test locally
  * `docker build -t docker-webserver-example .`
  * `docker run -d -p 80:80 --name docker-webserver-example -d docker-webserver-example`
  * Verify on http://localhost
  * `docker ps -a`
  * `docker rm -f docker-webserver-example`
* Commit and push to git repo
* Push image to dockerhub
  * https://docs.docker.com/engine/getstarted/step_six/
  * build and run image as described above
  * `docker images` - export DOCKER_IMAGE_ID
  * `export DOCKER_IMAGE_NAME=docker-webserver-example`
  * `export DOCKER_TAG="$(docker inspect -f '{{ .Created }}' $IMAGE_ID | sed -e 's/[:-]//g' | cut -d '.' -f 1)-$(git log -1 --pretty=format:"%h")"`
  * `export DOCKER_NAMESPACE=thewoolleyman`
  * `docker tag $DOCKER_IMAGE_ID $DOCKER_NAMESPACE/$DOCKER_IMAGE_NAME:$DOCKER_TAG`
  * `docker images` to verify
  * `docker push $DOCKER_NAMESPACE/$DOCKER_IMAGE_NAME`

### Manually run on Rancher

* Go to rancher UI
* Containers -> Add
  * Name
  * Image (include tag)
  * Port map: 80:80
  * Create

### Create pipeline manual steps

* Log into buildkite.com, create pipeline

----

## Various Gotchas

* AWS Security Group
  * SSH Access didn't work for Rancher AMI.  Cause was that default security group Inbound rule had source
    set to 'custom' value of the security group, instead of 'anywhere'.  TODO: Set up properly.
