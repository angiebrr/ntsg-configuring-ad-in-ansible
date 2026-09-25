# Configuring AD with Ansible

> [!WARNING]
> Archived and no longer maintained; kept for reference. Written in 2016 for CentOS 7, which is end-of-life, and it uses `always_run`, which Ansible removed in version 2.4.

- [Configuring AD with Ansible](#configuring-ad-with-ansible)
  - [Overview](#overview)
    - [What this playbook can do](#what-this-playbook-can-do)
  - [Using it](#using-it)
    - [Variable files](#variable-files)
      - [all.yml](#allyml)
      - [vault.yml](#vaultyml)
      - [defaults/ad.yml](#defaultsadyml)
      - [hosts](#hosts)
    - [Running the playbook](#running-the-playbook)
    - [Known issues](#known-issues)
      - [Metadata problems](#metadata-problems)
      - [Permission denied](#permission-denied)

## Overview

An Ansible playbook that joins CentOS 7 servers to an Active Directory domain with `realmd`, so people can log in with their AD account.

I wrote this in 2016 as the Linux sysadmin for NTSG, a research group at the University of Montana. It's the follow-up to [combining-ad-nis-ansible](https://github.com/angiebrr/combining-ad-nis-ansible), which kept NIS around for identities and made you create each computer in AD by hand first. 

**Tech:** Ansible, realmd, adcli, SSSD, Kerberos, oddjob-mkhomedir, chrony, CentOS 7

### What this playbook can do

- Set up an automatic creation of home directories upon login
- Configure the correct timezone and chrony settings
- Update your `sshd_config` file(s) to allow password authentication
- Join server(s) to an active directory domain (it will even provision your computer and add an entry for the computer in AD for you!)
- There is an option to only allow a certain number of AD groups to be able to log in
- There is an option to add certain groups as sudo users
- At the end of the playbook, it will gather all of the files that were backed up and put them in the backup directory on the remote server. By default, that is `/etc/backups`, but that can be changed if needed.

## Using it

### Variable files

You will need to do a little bit of setup before using this playbook. The first thing to do is to make sure that you have the following variable and host files filled out:

- `group_vars/all.yml`
- `defaults/vault.yml`
- `defaults/ad.yml`
- `hosts`

#### all.yml

This has the variables that either all hosts will use or will need to use. Edit `group_vars/all.yml` to match your setup. This file also has descriptions of each variable so you know exactly what you are setting.

```yaml
# group_vars/all.yml

# ------------------------------------------------------------------------
# ANSIBLE SPECIFIC VARS
# ------------------------------------------------------------------------

# Directory that will be created that's used to ensure idempotence for 
# things that were done that doesn't have built-in idempotence
playbook_metadata_dir: "/home/username/.ansible_metadata"

# ------------------------------------------------------------------------
# REALMD HOST NAME OVERRIDE
# ------------------------------------------------------------------------

# If you want to override the entire name of the computer explicitly, then
# include this variable. This should be done on a per-host basis and NOT in 
# this file (i.e. put it in group_vars/myhostname.yml).
# realmd_computer_name_override: "MYCOMPUTERNAME"

```

#### vault.yml

You will also need to create a vault.yml file with the following variables inside:

```yaml
# defaults/vault.yml

vault_ad_user: <AD username>
vault_ad_pass: <AD password>
```

It is recommended that you create this file using `ansible-vault` like so:

```bash
$ ansible-vault create defaults/vault.yml
```

For more information on using `ansible-vault`, see the Ansible documentation: [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

#### defaults/ad.yml

These variables should most definitely be modified to meet your active-directory needs. You will need to copy the template file, `ad.yml.example`, and create your custom file `ad.yml`. This file also has descriptions of each variable so you know exactly what you are setting.

```yaml
# defaults/ad.yml.example

# ------------------------------------------------------------------------
# AD DEFAULT VARIABLES
# ------------------------------------------------------------------------

# ------------------------------------------------------------------------
# NTP VARS
# ------------------------------------------------------------------------

timezone: "America/Boise"

# Whether or not you want to use the pool servers that come default with
# chrony, or if you want to specify an NTP server below in ntp_server.
use_default_ntp_servers: false

# Your preferred ntp servers. This has no effect if use_default_ntp_server
# is true.
ntp_servers: 
  - addr: 0.pool.ntp.org
  - addr: 1.pool.ntp.org

# ------------------------------------------------------------------------
# AD AND REALMD VARS
# See https://www.freedesktop.org/software/realmd/docs/realmd-conf.html
# ------------------------------------------------------------------------

# The active directory realm / domain. Will be used as the keroberos 
# domain as well.
ad_domain: AD.DOMAIN.COM

# (See defaults/ad.yml.example for the rest of the file)

```

#### hosts

This, as usual, contains the host information of the machines that will run this playbook (i.e. the inventory). There is a template file, `hosts.example`, that you can use if you wish.

### Running the playbook

A typical run of the playbook will look something like this:

```bash
$ ansible-playbook site.yml --ask-vault-pass
```

The `--ask-vault-pass` parameter will ask for the password to your vault-created file with the variables `vault_ad_user` and `vault_ad_pass`.

There are a couple of things to keep in mind when running this playbook: 
- Due to the necessity of using authconfig, adcli, and realmd (which can only be called from the command module), there are metadata directories used to ensure idempotence. If you manually change authconfig or run `realm permit`/`realm deny` yourself, see [Metadata problems](#metadata-problems).
- Every changed file is backed up in place, so do not despair if something goes awry! 

Here is a list of files that *could be* changed/created by the playbook:

- /etc/nsswitch.conf
- /etc/chrony.conf
- /etc/ssh/sshd_config
- /etc/realmd.conf
- /etc/krb5.conf
- /etc/sssd/sssd.conf

The only files that will be completely 100% clobbered (as opposed to just changing a few lines) are:
- /etc/realmd.conf

### Known issues

#### Metadata problems

If this playbook seems to not be changing something when it actually should be, be sure to first try to delete the `playbook_metadata_dir`. This directory was created because a couple of commands made it really difficult to test for idempotence (such as `authconfig` and `realm permit`), and so files were created that says, "Hey, these commands ran!" A potential problem with this is that, if `authconfig` or `realm permit` or `realm deny` get ran again, then the playbook's perception of what is changed and what isn't changed is incomplete. SO, if all else fails, delete the metadata directory to start on a "clean slate".

#### Permission denied

Another problem is if you get a "Permission Denied" when you know that the user has add/remove privileges and the password you put in `defaults/vault.yml` is correct, this may be due to the computer already existing but in a different OU than what you specified in `defaults/ad.yml`. This is fixed if you delete the computer in AD or if you move the computer where ansible is expecting it.
