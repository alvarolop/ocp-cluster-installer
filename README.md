# OCP cluster installer
This repository contains all what is necessary to install OCP 3.11 from scratch.
Clone the repo. All the resources needed in each section are in the folder with the same number.

<!-- TOC -->

1. [OCP cluster installer](#ocp-cluster-installer)
    1. [Section 1. Create VMs](#section-1-create-vms)
        1. [Kcli](#kcli)
        2. [Create the cluster with kcli](#create-the-cluster-with-kcli)
    2. [Section 2.Configure the VMs](#section-2configure-the-vms)
        1. [Directory structure](#directory-structure)
        2. [Configure 2-vms-configurer/group_vars/all.yml](#configure-2-vms-configurergroup_varsallyml)
        3. [Configure 2-vms-configurer/inventory](#configure-2-vms-configurerinventory)
        4. [Configure 2-vms-configurer/group_vars/vault.yml](#configure-2-vms-configurergroup_varsvaultyml)
        5. [Run the configurer](#run-the-configurer)
    3. [Section 3. Install OCP](#section-3-install-ocp)

<!-- /TOC -->

## Section 1. Create VMs

This example just defines three VMs in order to have the smaller cluster possible with all the functionality of a production cluster. All of them use a RHEL qcow2 image that will be subscribed in section 2. It deploys:
    - Bastion (bastion.miniocp.vm). Used to control OCP. This should be the only point to ssh from the laptop.
    - Master-infra (master.miniocp.vm). Big node with all the pods related to masters and infra nodes.
    - Node (node.miniocp.vm). Node that will host all the application pods.

### Kcli
I recommend using kcli to create the vms that will be part of the ocp cluster. This tool is meant to interact with existing virtualization providers (libvirt, kubevirt, ovirt, openstack, gcp and aws) and to easily deploy and customize vms from cloud images and also interact with those. Check the [Github project](https://github.com/karmab/kcli) and the [pdf](https://buildmedia.readthedocs.org/media/pdf/kcli/latest/kcli.pdf) of the documentation for more examples of configuration.

### Create the cluster with kcli
The following yaml shows the configuration of one VM using kcli:

```yaml
bastion.miniocp.vm:
  template: /home/alvaro/vms/rhel-server-7.6-x86_64-kvm.qcow2
  numcpus: 1
  memory: 4096
  disks:
    - size: 50
  nets:
    - name: default
      ip: 192.168.122.20
      mask: 255.255.255.0
      gateway: 192.168.122.1
  cmds:
    - echo root | passwd --stdin root
    - useradd -G wheel openshift
    - echo openshift | passwd --stdin openshift
```

Modify `template-rhel7.6-openshift-mini.yml` to set different values of cpu, memory, disks, network configuration or even add new nodes to the cluster. The kcli template also configures root/root and openshift/openshift. User `openshift` will be used to install OCP.


These are all the commands needed to create and manage VMs using kcli:

```bash
kcli plan -f template-rhel7.6-openshift-mini.yml openshift-mini # Create the OCP plan with name openshift-mini
kcli plan -s openshift-mini # Start the plan
kcli plan -w openshift-mini # Stop the plan
kcli list                   # Show VMs status
kcli plan -d openshift-mini # Delete the plan
```



## Section 2.Configure the VMs

This section uses Ansible to perform the following actions:

- In localhost:
  - Configure `/home/$USER/.ssh/config` to access VMs with the SSH correct user.
  - Configure `/etc/hosts` to resolve correctly the IPs of the VMs.
- In OCP VMs:
  - Copy the ssh keys.
  - Allow openshift users to escalate privileges without password.
  - Register the VMs.
  - Configure OCP repositories
  - Install and update packages.

### Directory structure

The following diagram shows the structure of folder "2-vms-configurer".

```bash
├── 2-vms-configurer
│   ├── ansible.cfg
│   ├── ansible.log
│   ├── group_vars
│   │   ├── all.yml
│   │   └── vault.yml
│   ├── inventory
│   ├── main.yml
│   └── named
│       ├── 122.168.192.db
│       ├── named.conf
│       └── ocp.vm.db
```

### Configure 2-vms-configurer/group_vars/all.yml

This file contains:
- All the hosts with IPs.
- Repositories to enable in the VMs.
- Packages to install in all the VMs and extra packages for the bastion.

Modify this yaml to add or remove any configuration


### Configure 2-vms-configurer/inventory

This file allows you to classify the hosts that you created in section 1. You can classify machines as you wish. The only requirement is that some packages will be installed only in VMs under the group `[bastion]`, so define at least one machine in this group.

For example, this is a copy of the file used currently:

```ini
[bastion]
bastion.miniocp.vm

[ocp]
master.miniocp.vm
node.miniocp.vm
```

### Configure 2-vms-configurer/group_vars/vault.yml

Create a new file `2-vms-configurer/group_vars/vault.yml` with the following content:

```yaml
rh_subscription_username: <RH_USERNAME>
rh_subscription_password: <RH_PASSWORD>
rh_subscription_poolid:   <POOL_ID>
```
This file has not been added to the repository for security purposes and `.gitignore` is configured to ignore this file.


### Run the configurer

Once all the files are set, prepare the nodes to host OCP is as simple as running the following command from the directory `2-vms-configurer`:

```bash
cd 2-vms-configurer
ansible-playbook main.yml --ask-vault-pass --ask-become-pass --ask-pass
```

This command will prompt you for three passwords required for different things:

1) **"SSH password"**. This password is prompted because the option `--ask-pass` is present. It allows you to connect to all the hosts to copy the ssh-key. By default, according to the kcli template, the password is `openshift`. This parameter is only necessary the first time that Ansible connects to each host.
   
2) **"SUDO password[defaults to SSH password]"**. This password is prompted because the option `--ask-become-pass` is present. It allows you to escalate privileges in your host machine to configure `/etc/hosts`. This is the password to run `sudo` commands.

3) **"Vault password"**. This password is prompted because the option `--ask-vault-pass` is present and allows you to open the vault file that contains information about your RH account.
   

## Section 3. Install OCP