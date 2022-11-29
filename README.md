# Cluster Installation Instructions

## Prerequisites for Cluster Installation

These instructions are based on the Installer-provisioned Infrastructure installation procedure.(<https://docs.openshift.com/container-platform/4.10/installing/installing_vsphere/installing-vsphere-installer-provisioned-customizations.html>)

1. Create an ssh-key pair for your user: <https://docs.openshift.com/container-platform/4.10/installing/installing_vsphere/installing-vsphere-installer-provisioned.html#ssh-agent-using_installing-vsphere-installer-provisioned>

2. Copy the contents of your ssh-key pub file to the last line, under `sshKey: |` in the following file:
  ansible/roles/initialize_cluster/templates/install-config.yaml.j2

3. Before installing the cluster, and finalizing steps, ensure the community.okd collection is installed:
`ansible-galaxy collection install community.okd`

4. Make sure there is a cluster specific variables folder and all.yaml under ansible/group_vars/cluster.
Use `ansible/group_vars/cluster/dev/all.yaml` as an example, changing variables to match new cluster.

5. New certificates will need to be saved in vaulted files using `ansible-vault` command: <https://docs.ansible.com/ansible/latest/user_guide/vault.html>
   1. Each cluster has a certs file. You can view the format of the file with `ansible-vault edit filename`
   2. Be sure to view the format of the existing certs files before adding a new one. If the format doesn't match the playbooks will fail. The certs info is indented four (4) spaces.
   3. Currently the vault password is being stored in plain text in resources/vault-password.txt. Suggest saving this file to an encrypted password application.
   4. Where you see `--vault-password-file=../resources/vault-password.txt` in the examples below, substitute with `--ask-vault-pass` for production builds, and should be a different password

## Initialize the Cluster

Ensure you are in the ansible directory before you run these commands

  ```sh
  ansible-playbook -i staging \
  -e @./group_vars/cluster/tst/all.yaml \
  -e @./group_vars/default-vault.yaml \
  -e @./group_vars/pull-secret.yaml \
  --vault-password-file=../resources/vault-password.txt \
  initialize_cluster.yaml
  ```

## Before Moving to Next Steps

Ensure you get the folder and template names for the new cluster you just built, and update the variables in the cluster specific all.yaml files

  ```sh
  cd install-dir
  grep folder *
  grep template *
  vi ../group_vars/cluster/[cluster]/all.yaml
  ```

Add the following to your .bashrc file so that KUBECONFIG environment variable set.
This file will be created, and updated, after you log into the cluster from the command line.

```sh
# User specific aliases and functions
export KUBECONFIG=~/.kube/config
```

## General instructions

### All Plays in One

This section shows the command to run all the plays as one big playbook.
Or go to the next section to run individual plays to update, or check a config setting.

```sh

    ansible-playbook -i staging \
    -e @./group_vars/cluster/[cluster]/all.yaml \
    -e @./group_vars/certs/cluster-[cluster]-certs.yaml \
    -e "cluster=[cluster]" \
    -e @./group_vars/vault-password.yaml \
    --vault-password-file=../resources/vault-password.txt \
    finalize_cluster.yaml

```

Each section below contains the commands for installing the individual plays to finalize a cluster.
You will need to replace the [cluster] defined in each with the cluster you are building (for instance: dev).

### Installing api and wildcard certs

  ```sh
    ansible-playbook -i staging certs_deployment_playbook.yaml \
    -e @./group_vars/certs/cluster-[cluster]-certs.yaml \
    -e "cluster=[cluster]" \
    --vault-password-file=../resources/vault-password.txt
  ```

### Create MachineConfigPool for Infra Nodes

  ```sh
  ansible-playbook -i staging mcp_playbook.yaml
  ```

### Installing machinesets

 When running this command you will be prompted for the type of machinesets to be created.
 Defaults to worker, but to build infra machinesets, type: `infra`

  ```sh
  ansible-playbook -i staging \
  -e @./group_vars/vault-password.yaml \
  -e @./group_vars/cluster/[cluster]/all.yaml \
  -e "cluster=[cluster]"
  --vault-password-file=../resources/vault-password.txt \
  machinesets_playbook.yaml
  ```

### Enable LDAP

  ```sh
  ansible-playbook -i staging \
  -e @./group_vars/vault-password.yaml --vault-password-file=../resources/vault-password.txt \
  ldap_playbook.yaml
  ```

### Enable LDAP Group Sync and role bindings

  ```sh
  ansible-playbook -i staging \
  -e  @./group_vars/vault-password.yaml --vault-password-file=../resources/vault-password.txt \
  ldap_groupsync_playbook.yaml
  ```

### Setting up Infra Taints and Tolerations

**Note**: Make sure all infra nodes are up and provisioned before running this step.

  ```sh
  ansible-playbook -i staging \
  -e @./group_vars/cluster/[cluster]/all.yaml \
  nodeconfig_playbook.yaml
  ```
