---
title: Production Media Server - Part I
updated: 2016-10-08 09:00
---

## Summary

I have the configuration management code for my [media server](https://github.com/skingry/chef.git) in a Github repository (aka 'infrastructure as code').  While there is documentation in the README.md, it only covers setting up the system for development purposes (ie. spinning up a Vagrant to do testing before committing new code to the repo).  With this article I intend on going over the basics I used to get a production system running using the same code repo.

This post will be broken down into two parts.  One going over the physical hardware and getting the Chef code running.  The other going over the networking concerns and application component setup/configuration.

## Hardware and Operating System Overview

For the [bare metal](https://en.wikipedia.org/wiki/Bare_machine), I am currently using a older [Supermicro workstation](https://www.supermicro.com/products/system/4U/7046/SYS-7046A-T.cfm) with a pair of [Xeon L5630](http://ark.intel.com/products/47927/Intel-Xeon-Processor-L5630-12M-Cache-2_13-GHz-5_86-GTs-Intel-QPI) quad-core processors, 48GB of ECC RAM, a 3Ware 9650SE 8 channel SATA RAID controller, 8x Western Digital RE3 1TB 7200RPM drives, a SanDisk 64GB SSD, and a Samsung 470 128GB SSD.

Each of the components were chosen using a matrix of cost, reliability, and performance.  Given the processors low TDP, the workstation wastes a minimal amount of power by way of heat loss (idle power consumption is ~170W, going to ~290W at full load) giving the system a very low operating cost (~$17 a month).  Each of the 1TB drives and controller were purchased used and are all 3Gb/s SATA, reducing their aquisition cost (the seller purchased and used them for less than 2000 hours before upgrading to a 6Gb/s controller and drives).  The combined read/write performace of the disks when in a ZFS array is enough to saturate my 1Gb/s network, so having 6Gb/s disks and controller would have been a waste of performance I would be unable to utilize.  Adding to that, the RE3 model has a >1.2M hour MTBF, 512 byte sector size (no ['ashift' settings](http://wiki.illumos.org/display/illumos/ZFS+and+Advanced+Format+disks) FTW!), low acoustic levels, and low power consumption as well... so using the older model didn't have any drawbacks in those areas.  I am using the 64GB SSD as a boot drive given it's sequential read performance and the 128GB SSD as a [ZFS L2ARC](https://blogs.oracle.com/brendan/entry/test) given it's blend of read, write, and durability characteristics.

The 1TB drives are setup to run in [JBOD](https://en.wikipedia.org/wiki/Non-RAID_drive_architectures#JBOD) mode via the RAID controller, allowing me to follow [best practices for setting up a ZFS file system](https://en.wikipedia.org/wiki/ZFS#ZFS_and_hardware_RAID).

The operating system I chose was [Ubuntu 14.04 LTS](http://releases.ubuntu.com/14.04/) for it's ease of use and robust documentation.  _Note: Plans to upgrade to [Ubuntu 16.04 LTS](http://releases.ubuntu.com/16.04/) have been made now that I have tested the installation of each of the application components inside the development environment using the new version of the OS and feeling satisfied that each are stable and operate reliably with one another._

## Setup

After installation of the OS and setup of an administation user (ie. `admin`), a `/data` storage array had to be defined.  Following the rough outline in the [`bootstrap.sh`](https://github.com/skingry/chef/blob/master/bootstrap.sh), I setup a RAID-Z2 array:

```
# apt-get -y install zfsutils-linux
# modprobe zfs
# zpool create -f tank raidz2 /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi
# zfs create -o mountpoint=/data tank/data
# zfs create -o mountpoint=/var/lib/docker tank/docker
# zpool add tank cache /dev/sdj
```

After running the above commands, I ended up with something like this:

```
# zpool status
  pool: tank
 state: ONLINE
config:

	NAME                                             STATE     READ WRITE CKSUM
	tank                                             ONLINE       0     0     0
	  raidz2-0                                       ONLINE       0     0     0
	    scsi-3600050e065bd330090100000af990000       ONLINE       0     0     0
	    scsi-3600050e065bd3800041c000088690000       ONLINE       0     0     0
	    scsi-3600050e065bd3800dda800008fdb0000       ONLINE       0     0     0
	    scsi-3600050e065bd3d0072a40000eb3d0000       ONLINE       0     0     0
	    scsi-3600050e065bd3d00a5d80000f9250000       ONLINE       0     0     0
	    scsi-3600050e065bd3d0094340000e4df0000       ONLINE       0     0     0
	    scsi-3600050e065bd4200e2bc0000b6dd0000       ONLINE       0     0     0
	    scsi-3600050e089275c0083100000e4df0000       ONLINE       0     0     0
	cache
	  ata-SAMSUNG_MZ7PA128HMCD-010L1_S0MUNEAC317704  ONLINE       0     0     0

errors: No known data errors
```

Next the I cloned the configuration management repo:

```
# git clone https://github.com/skingry/chef.git /chef
```

After cloning, a few files needed to be added:

```
# cd /chef
```

First, a new Chef solo config for the production setup:

```
# vi ./solo/config/production.rb
```

```
node_name                  "monolith.robotozon.com"

file_cache_path            "/chef/cache"
file_backup_path           "/chef/backup"

log_level                  :info
verbose_logging            false

environment                "production"

cookbook_path              [ "/chef/cookbooks", "/chef/site-cookbooks" ]
environment_path           "/chef/environments"
role_path                  "/chef/roles"
data_bag_path              "/chef/data_bags"
```

Next, I added a new environment to match the solo config:

```
# vi ./environments/production.json
```

```
{
  "name": "production",
  "description": "Production Environment",
  "json_class": "Chef::Environment",
  "chef_type": "environment",
  "cookbook_versions": {
  },
  "default_attributes": {
    "cron_mailto": "sjkingry@gmail.com",
    "set_fqdn": "*.robotozon.com",
    "media_server": {
      "domain": "robotozon.com",
      "s3_bucket": "s3.robotozon.com"
    }
  },
  "override_attributes": {
  }
}
```

Then I created the adminstrative user's data bag (the username used here matches the one of the user I created during the installation of the operating system):
*(NOTE: the `htpasswd` is a hash of a plaintext password generated using an [Apache specific method](https://httpd.apache.org/docs/2.4/misc/password_encryptions.html).  The hash can be generated using the following command, replacing 'PASSWORD' with the plaintext password: `openssl passwd -apr1 PASSWORD`)*

```
# vi ./data_bags/users/admin.json
```

```
{
  "id": "admin",
  "ssh_keys": [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN6eh14uv+99WlYv6pM1xWv70Sjgx1Mt2GimKwtR+xqv"
  ],
  "groups": [
    "sysadmin"
  ],
  "uid": 1000,
  "htpasswd": "$apr1$NvuTjET7$0yqS//xIYezm1K.OuXD7u1",
  "shell": "\/bin\/bash",
  "comment": "System Administrator"
}
```

Next I installed the prerequisites needed by Chef:

```
# apt-get -y install build-essential curl dialog git python ruby ruby-dev wget
# gem install --no-ri --no-rdoc chef -v 12.9.38
# gem install --no-ri --no-rdoc berkshelf -v 4.3.2
# gem install --no-ri --no-rdoc bundler
```

Finally, I ran Chef using the new production solo config to complete the installation:

```
# cd /chef
# berks vendor cookbooks
# chef-solo -c /chef/solo/config/production.rb -j /chef/solo/json/server.json
```

That concludes part one of this blog post.
