# continuous-everything

## Overview

My attempt to automate as much of my hacking activity as possible - tests, deployment, etc.

Once I commit and push, everything else should happen automatically. 

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
