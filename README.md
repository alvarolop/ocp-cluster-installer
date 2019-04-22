# ocp_cluster_installer
This repo contains all what is necessary to install OCP 3.11 from scratch.

## Create VMs and configure them

### Kcli
I recommend using kcli to create the vms that will be part of the ocp cluster. This tool is meant to interact with existing virtualization providers (libvirt, kubevirt, ovirt, openstack, gcp and aws) and to easily deploy and customize vms from cloud images and also interact with those. Check the [Github project](https://github.com/karmab/kcli) and the [pdf](https://buildmedia.readthedocs.org/media/pdf/kcli/latest/kcli.pdf) of the documentation for more examples of configuration.

## Create the cluster with kcli
```bash
kcli plan -f template-rhel7.6-openshift-mini.yml openshift-mini # Create the plan
kcli plan -s openshift-mini # Start the plan
kcli plan -w openshift-mini # Stop the plan
kcli list                   # Show VMs status
kcli plan -d openshift-mini # Delete the plan
```


ansible-playbook main.yml --ask-vault-pass -K --ask-pass
Where:
--ask-vault-pass => Contraseña del vault
--ask-pass => Contraseña para ssh con contraseña (Solo necesario si se ejecuta por primera vez la tarea "Copy ssh keys")
--ask-become-pass => Contraseña para escalar privilegios en local. Se usa en la máquina local solo. 