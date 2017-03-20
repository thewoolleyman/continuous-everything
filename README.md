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

## Rancher + Spotinst on AWS

* Set up Rancher on AWS: https://docs.rancher.com/os/running-rancheros/cloud/aws/

### AWS One-Time Setup For Rancher + Spotinst
  * Create 'rancher' IAM user and group with full Administrator access, save access key in lastpass
  * Set up AWS CLI locally: `brew install awscli`
  * `aws configure`
  * Create Keypair:
    * `aws ec2 create-key-pair --region us-west-2 --key-name rancher` - save in lastpass
  * Create and tag VPC:
    * `aws ec2 create-vpc --cidr-block '10.0.0.0/16'`
    * `aws ec2 create-tags --resource $VPC_ID --tags Key=Name,Value=rancher`
  * Find and Name Network ACL:
    * `aws ec2 describe-network-acls --filters Name=vpc-id,Values=$VPC_ID`
    * `aws ec2 create-tags --resource $ACL_ID --tags Key=Name,Value=rancher`
  * Create and Name Security Group:
    * `aws ec2 create-security-group --group-name rancher --description rancher --vpc-id=vpc-567e3831`
    * `aws ec2 create-tags --resource sg-50cddb28 --tags Key=Name,Value=rancher`
  * Create, Attach, and Name Internet Gateway:
    * `aws ec2 create-internet-gateway`
    * `aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID`
    * `aws ec2 create-tags --resource $IGW_ID --tags Key=Name,Value=rancher
  * Find and Name Route Table:
    * `aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC_ID`
    * `aws ec2 create-tags --resource $RTB_ID --tags Key=Name,Value=rancher`
  * Add route for Internet Gateway
    * `aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID`
  * Create, Associate, and Name Subnet:
    * `aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block '10.0.1.0/24' --availability-zone us-west-2a`
    * `aws ec2 associate-route-table --subnet-id $SUBNET_ID --route-table-id $RTB_ID`
    * `aws ec2 modify-subnet-attribute --map-public-ip-on-launch --subnet-id $SUBNET_ID`
    * `aws ec2 create-tags --resource $SUBNET_ID --tags Key=Name,Value=rancher`
  * Create and Name EBS Volume to hold persistent Rancher Mysql DB data
    * `aws ec2 create-volume --volume-type gp2 --size 2 --availability-zone us-west-2a`
    * `aws ec2 create-tags --resource $VOL_ID --tags Key=Name,Value=rancher`
  * Create AWS route53 Public Hosted Zone and Get Zone ID
    * https://console.aws.amazon.com/route53/home#hosted-zones: - name after purchased $HOSTED_ZONE_NAME
    * `aws route53 list-hosted-zones-by-name --dns-name $HOSTED_ZONE_NAME`
    * `aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol all --cidr 0.0.0.0/0`
    * `aws ec2 authorize-security-group-egress --group-id $SG_ID --protocol all --cidr 0.0.0.0/0`

### AWS steps to Recreate RancherOS server instance
  * Create and Name RancherOS server instance:
    * RancherOS AMI for us-west-2 (Oregon) region: https://github.com/rancher/os/blob/master/README.md
    * `aws ec2 run-instances --region us-west-2 --image-id $AMI_ID --count 1 --instance-type t2.micro --subnet-id $SUBNET_ID --security-group-ids $SG_ID --key-name rancher`
    * `aws ec2 create-tags --resource $I_ID --tags Key=Name,Value=rancher`    
  * Attach MYSQL EBS volume:
    * `aws ec2 attach-volume --volume-id $VOL_ID --instance-id $I_ID --device /dev/sdm`
  * Assign DNS A record to instance's public IP:
    * Get Public IP Address: `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=$VPC_ID`
    * `aws route53 change-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID --change-batch "{\"Changes\":[{\"Action\":\"UPSERT\",\"ResourceRecordSet\":{\"Name\":\"$RECORD_SET_NAME\",\"Type\":\"A\",\"TTL\":300,\"ResourceRecords\":[{\"Value\":\"$PUBLIC_IP_ADDRESS\"}]}}]}"`
* https://console.spotinst.com/#/wizard/aws
* 


## Various Gotchas

* AWS Security Group
  * SSH Access didn't work for Rancher AMI.  Cause was that default security group Inbound rule had source
    set to 'custom' value of the security group, instead of 'anywhere'.  TODO: Set up properly.
