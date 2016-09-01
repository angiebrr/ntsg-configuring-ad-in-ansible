# Configuring AD with Ansible

## Overview

This playbook joins a machine to an AD domain. 

So far, only `CentOS 7` machines are supported in this playbook because of the reliance of `realmd`. The playbook may be updated in the future to support `CentOS 6` machines, as you can use `adcli` to join machines to active directory, but this is much more of a manual process.

## Variable Files

You will need to do a little bit of setup before using this playbook. The first thing to do is to make sure that you have the following variable and host files filled out:

- `group_vars/all.yml`
- `group_vars/vault.yml`
- `roles/ad/defaults/main.yml`
- `hosts`

### all.yml

This has the variables that either all hosts will use or will need to use. You will need to copy the template file, `all.yml.example`, and create your custom file `all.yml`. This file also has descriptions of each variable so you know exactly what you are setting. 

```yaml
# group_vars/all.yml.example

# ------------------------------------------------------------------------
# ANSIBLE SPECIFIC VARS
# ------------------------------------------------------------------------

# User that ansible will use to log in with
playbook_remote_user: username

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

# ------------------------------------------------------------------------
# VAULT VARS
# ------------------------------------------------------------------------
# Be sure to include these variables in your own group_vars/vault.yml file
# Note that this user needs to be able add/remove computers
# vault_ad_user: <AD username>
# vault_ad_pass: <AD password>

```

### vault.yml

You will also need to create a vault.yml file with the following variables inside:

```yaml
# group_vars/vault.yml

vault_ad_user: <AD username>
vault_ad_pass: <AD password>
```

It is recommended that you create this file using `ansible-vault` like so:

```bash
$ ansible-vault create group_vars/vault.yml
```

For more information on using `ansible-vault`, please visit this the ansible documentation: [Ansible Vault](http://docs.ansible.com/ansible/playbooks_vault.html "Ansible's Documentation for Vault") 

### ad/defaults/main.yml

These variables should most definitely be modified to meet your active-directory needs. You will need to copy the template file, `main.yml.example`, and create your custom file `main.yml`. This file also has descriptions of each variable so you know exactly what you are setting.

```yaml
# roles/ad/defaults/main.yml.example

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

# (See ad/defaults/main.yml for the rest of the file)

```

### hosts

This, as usual, contains the host information of the machines that will run this playbook (i.e. the inventory). There is a template file, `hosts.example`, that you can use if you wish.

## Provision the machine(s) in hosts

Before running this playbook, you have to provision the machines in your hosts file in Active Directory in Windows, as I haven't found a way to automate this. All you need to do is add a computer under the AD domain and make it the same name as the hostname of the linux machine.

## Running the playbook

A typical run of the playbook will look something like this:

```bash
$ ansible-playbook site.yml --ask-vault-pass
```

The `--ask-vault-pass` parameter will ask for the password to your vault-created file with the variables `vault_ad_user` and `vault_ad_pass`.

There are a couple of things to keep in mind when running this playbook: 
- Due to the necessity of using authconfig and adcli (which are can be only called from the command module), running the playbook in its entirety will always return a "changed" of 3. Later versions of this playbook will strive to change that behavior.
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

## Unexpected Behavior

If this playbook seems to not be changing something when it actually should be, be sure to first try to delete the `playbook_metadata_dir`. This directory was created because a couple of commands made it really difficult to test for idempotence (such as `authconfig` and `realm permit`), and so files were created that says, "Hey, these commands ran!" 

A potential problem with this is that, if `authconfig` or `realm permit` or `realm deny` get ran again, then the playbook's perception of what is changed and what isn't changed is incomplete. SO, if all else fails, delete the metadata directory to start on a "clean slate".