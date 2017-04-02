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
* Tweak CloudWatch rules (CloudWatch -> Alarms -> Check one -> Actions -> Modify):
  * ~~scale down after 5 minutes instead of 30~~
  * ~~scale up after 5 minutes instead of 1~~
* Tweak EC2 Autoscaling Group 
  (EC2 -> Auto Scaling -> Auto Scaling Groups -> Check buildkite one -> Actions -> Edit -> Scaling policies)
  * AgentScaleUpPolicy -> Actions -> Edit
  * Wait 360 seconds instead of 120 before allowing another scaleup activity (give agents time to start and run builds)


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
* Allocate public elastic IP for rancher server
  * `aws ec2 allocate-address --domain vpc`
  * `aws ec2 describe-addresses` # provides $ALLOCATION_ID and $PUBLIC_IP_ADDRESS

### AWS steps to Recreate RancherOS server instance

* Set env vars from one-time setup:
  * `source ./set-env-vars`
* Create and Name RancherOS server instance:
  * RancherOS AMI for $REGION region: https://github.com/rancher/os/blob/master/README.md
  * `aws ec2 run-instances --region $REGION --image-id $AMI_ID --count 1 --instance-type t2.small --subnet-id $SUBNET_ID_1 --security-group-ids $SG_ID --key-name rancher`
  * `aws ec2 create-tags --resource $I_ID --tags Key=Name,Value=rancher-server`
* Assign elastic IP
  * `aws ec2 associate-address --instance-id $I_ID --allocation-id $ALLOCATION_ID`
* Attach MYSQL EBS volume:
  * `aws ec2 attach-volume --volume-id $VOL_ID --instance-id $I_ID --device /dev/xvdm`
* Assign DNS A record to instance's allocated public elastic IP:
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


## Install Rancher CLI

https://docs.rancher.com/rancher/v1.2/en/cli/

### Install

* Create a new Account API key for `cli`, save in lastpass
* Download and install CLI by clicking in lower-right corner of Web UI.
* Unzip and put executable in `/usr/local/bin`
* `rancher config`
  * URL: `http://rancher.$HOSTED_ZONE_NAME:8080`
  * Access/Secret key created for `cli` above

### Test various CLI commands

Note: Some may be blank or not work until containers/services are created in subsequent steps

* `rancher hosts`
* `export RANCHER_DOCKER_HOST=<HOSTNAME of host>` or use `--host`
  * Note this is the *internal* hostname, not the external hostname or IP.
* `rancher docker ps`
* `rancher ps`
* `rancher ps -c`
* `rancher exec -i -t <container ID or name from rancher ps> /bin/bash`


## Add Rancher Route53 Infrastructure Stack

* Stacks -> Add Infrastructure -> Add from catalog -> Route 53
* Enter rancher aws key info
* Enter $HOSTED_ZONE_NAME
* Launch
* Verify (after a stack is created below)
  * Verify it looks right in AWS route53 UI.
  * Verify - go to $RANCHER_LB_NAME, info -> details, verify web server served at FQDN.


## Manually create example docker image

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

## Manual Set up of Example Stack with Load Balancing in Rancher via UI

### Add a stack
* Rancher UI - Stacks - Add Stack
* Stack Name: 'example-stack' - Create

### Add Webserver service
* Add Service
  * ~~ZERO containers (it will be started via load balancer)~~ 1 container
  * Name 'example-webserver'
  * Image (include tag, e.g.: `thewoolleyman/docker-webserver-example:20170323T035310-4d57408`)
  * Port map: Public Host Port BLANK (for random), Private Container Port 80 (exposed by container)
  * Create
  
### Add Load Balancer
* In Rancher stack, click dropdown by "Add Service" and "Add load balancer"
* LB Name: 'example-load-balancer'
* Run 1 container
* Port Rules: Public, HTTP, Request Host Port 80, target webservice 'example-webserver', Port 80
* Create
* Verify:
  * Go to example stack
  * Wait until LB started
  * Click "i" for info next to name
  * Click "View Details"
  * Ports tab
  * navigate to Host IP and verify it serves the example webserver html.

## Create a shorter custom domain name for a Rancher LB

* Create a shorter custom Route53 CNAME to point to default template one autocreated via Route53.
  * `source set-env-vars`
  * `aws route53 change-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID --change-batch "{\"Changes\":[{\"Action\":\"UPSERT\",\"ResourceRecordSet\":{\"Name\":\"www-example.$HOSTED_ZONE_NAME\",\"Type\":\"CNAME\",\"TTL\":300,\"ResourceRecords\":[{\"Value\":\"$RANCHER_LB_NAME.$COMPOSE_PROJECT_NAME.$RANCHER_ENV.$HOSTED_ZONE_NAME.\"}]}}]}"`

## Automated Set up of Example Stack with Load Balancing in Rancher via Rancher Compose CLI

* Review docs:
  * https://docs.docker.com/compose/
  * https://docs.rancher.com/rancher/v1.5/en/cattle/rancher-compose/
  * https://docs.rancher.com/rancher/v1.1/en/cattle/adding-services/
* Download compose CLI by clicking bottom right of Rancher UI
* install to `/usr/local/bin`
* Create env API key:
  * Go to Rancher UI -> API -> Keys -> Advanced -> Add Environment API Key
  * Name `rancher-compose-<ENV>`, where ENV is `default` save in lastpass
* Setup API keys
  * `mkdir -p ~/.rancher/default`
  * `vi ~/.rancher/default/api-keys`
    * `RANCHER_ACCESS_KEY=XXX`
    * `RANCHER_SECRET_KEY=YYY`
* `source set-env-vars`
* `rancher-compose -e ~/.rancher/default/api-keys -r rancher/$RANCHER_ENV/$COMPOSE_PROJECT_NAME/rancher-compose.yml up -d -u`

## BuildKite Example Continuous Delivery Pipeline Example Manual Setup

* Log into buildkite.com

### Create docker-webserver-example pipeline

* Create pipeline `docker-webserver-example`
  * repo URL
  * Add step to read steps from pipeline
* Set up webhook
  * https://buildkite.com/continuous-everything/docker-webserver-example/settings/setup/github
  * Follow instructions
* Add secrets
  * https://github.com/buildkite/elastic-ci-stack-for-aws#build-secrets
  * https://buildkite.com/docs/agent/hooks#creating-hook-scripts
  * Generate `private_ssh_key` requires for buildkite secrets
    * `ssh-keygen -t rsa -b 4096 -f id_rsa_buildkite`
    * save in lastpass
    * upload using s3 CLI command in above links 
  * Create temp files for following hook scripts according to example in above links, upload to `continuous-everything-buildkite-secrets`
    bucket.  Should contain the following scripts/vars:
    * 
    * `/env`
      * asdf

----

## Various Tips/Gotchas

### Tips

* Using TCPDump to debug network issues:
  * https://danielmiessler.com/study/tcpdump/#gs.YMVi1ds
* Debugging containers
  * use rancher CLI ssh to ssh to container
  * `apt-get update`
  * `apt-get install -y <package>` - e.g. telnet, wget, tcpdump

### Gotchas

* AWS Security Group
  * SSH Access didn't work for Rancher AMI.  Cause was that default security group Inbound rule had source
    set to 'custom' value of the security group, instead of 'anywhere'.  TODO: Set up properly.
* Use random ports for containers, let Rancher manage them.
* Simple HTML page hosts may be cached by browser.  Use dev tools to explicitly clear cache or add a 
  url param.  More info: https://css-tricks.com/strategies-for-cache-busting-css/
* Use google DNS server (8.8.8.8), local DNS servers may not update by the TTL
