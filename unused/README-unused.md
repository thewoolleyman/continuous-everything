# Unused

The `unused` folder contains aborted/incomplete/unused notes, config files, etc., which are not used by
the currently running environment, but may still be useful for future reference.

----

# Rimu

## Docker Host

Hosted at https://rimuhosting.com/

### Docker Host Manual Steps

* Add SSH public key via Rimu control panel

#### CoreOS
* ~~Install VPS with CoreOS~~: https://blog.rimuhosting.com/2015/09/04/rimuhosting-offering-coreos/
* **NOTE: Didn't end up using CoreOS because Rimu's distro only had docker 1.11, which doesn't support Rancher**
* Use cloud-init checked into this repo
  * TODO: try to automate via api:
  * https://blog.rimuhosting.com/2009/10/20/rimuhostings-rest-ful-api/
  * http://apidocs.rimuhosting.com/jaxrsdocs/index.html
  * https://github.com/apache/libcloud/blob/trunk/libcloud/compute/drivers/rimuhosting.py
  * https://blog.rimuhosting.com/2012/12/04/custom-vps-image-via-api/

#### Ubuntu Docker Host

* Install Rimu Ubuntu 16.04 LTS 64-bit VPS
* `ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@72.249.37.36`
* 

## Install Rancher Manual Steps

* ssh to docker host - https://docs.rancher.com/rancher/v1.5/en/quick-start-guide/
* `sudo docker run -d --restart=unless-stopped -p 8080:8080 rancher/server`
* set up access control: https://docs.rancher.com/rancher/v1.5/en/configuration/access-control/

