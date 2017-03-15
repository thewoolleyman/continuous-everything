# continuous-everything

## Overview

My attempt to automate as much of my hacking activity as possible - tests, deployment, etc.

For anything I need to do more than once, once I commit and push, everything else should happen automatically. 

I will attempt to list everything I had to do manually here, even if it's not in complete detail.
 
## Buildkite

The starting point for automation of test/build/delivery pipelines.

### Buildkite Manual Steps

* Get an account on https://buildkite.com
* Create an Org
* Create an Agent, select "AWS" type, and follow their instructions to set up
  the [Elastic CI Stack for AWS](https://github.com/buildkite/elastic-ci-stack-for-aws)
  * Take their suggestions to "Follow best practice by setting up a separate development AWS account and using
    role switching and consolidated billing"

## Docker Host

Hosted at https://rimuhosting.com/

### Docker Host Manual Steps

* Add SSH public key via control panel
* Install VPS with CoreOS: https://blog.rimuhosting.com/2015/09/04/rimuhosting-offering-coreos/
  * TODO: try to automate via api:
  * https://blog.rimuhosting.com/2009/10/20/rimuhostings-rest-ful-api/
  * http://apidocs.rimuhosting.com/jaxrsdocs/index.html
  * https://github.com/apache/libcloud/blob/trunk/libcloud/compute/drivers/rimuhosting.py
  * https://blog.rimuhosting.com/2012/12/04/custom-vps-image-via-api/


