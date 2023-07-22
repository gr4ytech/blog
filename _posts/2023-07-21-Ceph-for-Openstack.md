---
layout: post
date: 2023-07-21 16:00:00 -0600
title: Ceph for Openstack
categories: [Ceph]
tags: [ceph,cloud,openstack]
---

## Introduction

Getting Ceph functional for an Openstack deployment can be difficult, even if you are using an automated deployment tool like Kolla Ansible. While resources exist, not everything will work first try with Openstack. In my experience, the integration of Ceph and Openstack proved to be more troublesome than the initial deployment of either platform. 

Most of the resources here are based on personal experience or the standard documentation instructions from Ceph. This guide will be geared towards integrating Ceph with Kolla Ansible.

## Prepping Ceph for Cinder and Glance 

Before you're able to get Ceph and Openstack talking with each other, you need to do the initial prep for Ceph by creating pools and users for RBD. For pools, it is highly recommended to not use autoscaling and instead use the PG Calc utility to figure out how many Placement Groups you need. From there you can tell Ceph to warn you if the pool is getting full. You can find the PG Calc tool in [Resources](#resources).

Before you can create users, you need to create pools for RBD. For a typical Cinder/Glance config, you would need either 2 or 3 pools depending on if you have Cinder backup functionality enabled. We will be assuming the use of Cinder, Cinder Backup, and Glance for this tutorial. If you require ephemeral storage, include a 4th pool for ephemeral vms.

### Install the Ceph dependencies



### Creating Ceph Pools

To create the pools, you can either use the Ceph dashboard or Ceph CLI. I would suggest creating the pools through the CLI if you need custom crush rules (you have both hard drives and SSDs but you only want SSDs for VMs). 

From the GUI: 
1. Click on the "Pools" section after logging in.
2. Click the plus with "Create Pool"
3. Enter the name of the pool
4. Select replicated as the type of pool.
5. Turn PG Autoscale to Warn and enter the number of placement groups indicated by the PG Calc tool.
6. Add RBD as the Application
7. Under the CRUSH section, select the rule you want to use for the pool.

> Create a new CRUSH rule here if you need to separate the types of OSDs you have selected. We have the Glance pool striped across all OSDs, Cinder Volumes across SSDs, and Cinder Backups across only Hard Drives.
{: .prompt-info }

After you've decided on what options for either compression or quotas you need for the pool, click "Create Pool" if you're happy with the current settings. Repeat these steps for all the pools that you need. 

If you want to create everything from the CLI, I would highly recommend creating a crush rule from the GUI or learning the format for how to make a crush rule. First, create the pool with the flags you need:

> To do any changes to Ceph from CLI, you need to be on a node that has the ``<clustername>.client.admin.keyring``{: .filepath } keyring file.
{: .prompt-tip }

```bash
ceph osd pool create <pool-name> <pg-num> replicated \
         <crush-rule-name> --autoscale-mode=warn
```

Afterwards enable rbd on the pool you created:

```bash
rbd pool init <pool-name>
```

### Creating Ceph User Accounts

The next step is to create Ceph users for Cinder, Cinder backup, and Glance.

First is Glance. Replace the ``rbd pool=<glance-pool-name>`` with the name of your glance pool you made earlier:
```bash
ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=<glance-pool-name>' mgr 'profile rbd pool=<glance-pool-name>'
```

Next is the Cinder user. This has more permissions and can be used to access both cinder volumes and nova ephemeral storage if you have that enabled. If you are not using ephemeral storage follow this command:
```bash
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=<cinder-pool-name>, profile rbd-read-only pool=<glance-pool-name>' mgr 'profile rbd pool=<cinder-pool-name>'
```

Otherwise follow this command:
```bash
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=<cinder-pool-name>, profile rbd pool=<nova-pool-name>, profile rbd-read-only pool=<glance-pool-name>' mgr 'profile rbd pool=<cinder-pool-name>, profile rbd pool=<nova-pool-name>'
```

> Remember to replace the pool name placeholders with the ones you created! 
{: .prompt-tip }

Finally is the Cinder Backup user. This user has access to the Cinder backup pools so it can properly do its job. Replace the ``<cinder-backup-pool-name>`` with the ones you made:
```bash
ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=<cinder-backup-pool-name>' mgr 'profile rbd pool=<cinder-backup-pool-name>'
```

Congratulations! All of the Ceph configuration required to make the Openstack integration work is complete! 

### Export Ceph configs

Now you need to export the keyrings for the users which can be done here:
```bash
ceph auth export client.<name> -o "<clustername>.client.<name>.keyring"
```

> You can remove the ``caps`` entries from the keyring after exporting the file.
{: .prompt-tip }

And the ceph configuration file:
```bash
ceph config generate-minimal-conf > <minimal-config-path>
```

> The spaces in the ``ceph.conf`` file need to be removed before adding to the Kolla config file structure.
{: .prompt-tip }

## Adding Config files to Kolla Ansible

While Kolla Ansible will be explained in more detail, the basic file structure for the config files for Ceph are like so:
```shell
.
├── cinder
│   ├── ceph.conf
│   ├── cinder-backup
│   │   ├── ceph.client.cinder-backup.keyring
│   │   └── ceph.client.cinder.keyring
│   └── cinder-volume
│       └── ceph.client.cinder.keyring
├── cinder.conf
├── glance
│   ├── ceph.client.glance.keyring
│   └── ceph.conf
├── glance-api.conf
├── nova
│   ├── ceph.client.cinder.keyring
│   └── ceph.conf
└── nova.conf
```

Each folder represents the custom config for each Openstack service you need to configure, and for Ceph, that includes Cinder, Glance, and Nova. The associated custom configurations with Ceph tweaks for Cinder, Glance, and Nova are in the cinder.conf, glance-api.conf, and nova.conf files respectively. 

> Even if you are not using Nova ephemeral storage, you still need to add the cinder client keyring and ceph.conf file for compute nodes to function properly.
{: .prompt-info }

### Tweaks for Openstack with Ceph

These are some custom Openstack config files that get merged into the existing ones that are built by Kolla during deployment. These include several tweaks so Ceph can run better with the existing Openstack components. Create new config files for the associated services in the ``/etc/kolla/config``{: .filepath } folder but *not* in the services folder. Follow the directory tree above for guidance.

#### Cinder

In the ``cinder.conf`` file, by default, Openstack will timeout a Ceph connection after 5 seconds if nothing is received. This can cause issues with creating volumes for VMs. I recommend that this feature gets turned off by default. 

```
[rbd-1]
rados_connect_timeout = -1
rbd_flatten_volume_from_snapshot = false
```

#### Glance

In the ``glance-api.conf`` file, there is no copy-on-write (COW) turned on by default. This can cause issues with virtual machines timing out on the build process because it is unable to retrieve the Glance image in time. COW exposes the glance backend, so if for some reason Ceph is not the only backend being used to store images, this setting should remain off.

```
[DEFAULT]
show_image_direct_url = True
```

#### Nova

In the ``nova.conf`` file, the timeout for creating a new Cinder volume is incredibly short by default. This causes any virtual machine image that is over 10GB in size to immediately fail during the build process. Increase the timeout for creating the volumes to avoid headaches later. 

```
[DEFAULT]
block_device_allocate_retries = 300
block_device_allocate_retries_interval = 3
```

## Pitfalls

### Openstack shows Cinder (Backup) service is Down

Ensure that the ``ceph.conf`` and the cinder/cinder-backup keyrings are in the ``/etc/ceph`` directory. Also check that the keyrings are able to talk to the Ceph cluster. 


### Nova is unable to launch instances

If everything else with Ceph is functional (Glance/Cinder services are up and working) You may need to change the UUID secret for cinder.

Grab a new RBD uuid with:

```bash
uuidgen
```

Then replace the ``cinder_rbd_secret_uuid`` value in the ``passwords.yml`` file. After this, on the Kolla deployment node run:

```bash
kolla-ansible -i <inventory> reconfigure
```

This will reconfigure all of the Kolla Openstack containers that are deployed with the specific inventory file. To speed up the process, it might be advisable to only include the compute, networking, and control nodes in the inventory file.

### 

## Resources

These are some of the resources I used while getting Openstack and Ceph working together:

- [Ceph Placement Group Calculator](https://old.ceph.com/pgcalc/)
- [Block Devices and Openstack Ceph Documentation](https://docs.ceph.com/en/latest/rbd/rbd-openstack/)
- [Crush Rules (Doesn't require RedHat account)](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/1.2.3/html/storage_strategies/crush-rules)