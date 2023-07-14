# Cluster Installation Instructions

- [Cluster Installation Instructions](#cluster-installation-instructions)
  - [Prerequisites for Cluster Installation](#prerequisites-for-cluster-installation)
  - [Initialize the Cluster](#initialize-the-cluster)
  - [Before Moving to Next Steps](#before-moving-to-next-steps)
  - [General instructions](#general-instructions)
    - [All Plays in One](#all-plays-in-one)
      - [Install Certs First](#install-certs-first)
      - [Finalize Cluster](#finalize-cluster)
      - [Building Quay Cluster](#building-quay-cluster)
      - [Install ArgoCD Projects](#install-argocd-projects)
  - [Day 2: Post Initialization Instructions (Individual Playbooks)](#day-2-post-initialization-instructions-individual-playbooks)
    - [Installing api and wildcard certs](#installing-api-and-wildcard-certs)
    - [Include Chrony Machineconfigs](#include-chrony-machineconfigs)
    - [Installing machinesets](#installing-machinesets)
    - [Enable LDAP](#enable-ldap)
    - [Enable LDAP Group Sync and role bindings](#enable-ldap-group-sync-and-role-bindings)
    - [Setting up Infra Taints and Tolerations](#setting-up-infra-taints-and-tolerations)
    - [Configure OperatorHub to Pull from Disconnected Quay](#configure-operatorhub-to-pull-from-disconnected-quay)
    - [Configure Update Services to pull from Disconnected Quay](#configure-update-services-to-pull-from-disconnected-quay)
    - [Install an Operator](#install-an-operator)
      - [Example of grep for Operator Information](#example-of-grep-for-operator-information)
      - [Run playbook](#run-playbook)
      - [Configs for Operators](#configs-for-operators)
    - [Toggle Operator Update Mode](#toggle-operator-update-mode)
    - [Install ArgoCD](#install-argocd)
    - [Install Bitnami](#install-bitnami)
    - [Deploy Network Policy Template](#deploy-network-policy-template)
    - [Create ArgoCD Projects](#create-argocd-projects)
    - [Configure Image Registry](#configure-image-registry)
    - [Move Monitoring Pods to Infra Nodes](#move-monitoring-pods-to-infra-nodes)
  - [Update Pull Secret](#update-pull-secret)
  - [Backup of Bitnami Sealed Secret](#backup-of-bitnami-sealed-secret)

## Prerequisites for Cluster Installation

These instructions are based on the Installer-provisioned Infrastructure installation procedure.(<https://docs.openshift.com/container-platform/4.10/installing/installing_vsphere/installing-vsphere-installer-provisioned-customizations.html>)

1. Create an ssh-key pair for your user: <https://docs.openshift.com/container-platform/4.10/installing/installing_vsphere/installing-vsphere-installer-provisioned.html#ssh-agent-using_installing-vsphere-installer-provisioned>

2. Copy the contents of your ssh-key pub file to the last line, under `sshKey: |` in the following file:
  ansible/roles/initialize_cluster/templates/install-config.yaml.j2

3. Update group_vars/pull-secret.yaml with customer pull-secret
`ansible-vault edit group_vars/pull-secret.yaml --vault-password-file=../resources/vault-password.txt`

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
ansible-playbook -i inventory \
-e @./group_vars/cluster/[cluster]/all.yaml \
-e @./group_vars/env/[env]/default-vault.yaml \
-e @./group_vars/env/[env]/all.yaml \
-e @./group_vars/pull-secret.yaml \
-e @./group_vars/cluster/[cluster]/ssh-key.yaml \
--vault-password-file=../resources/vault-password.txt \
initialize_cluster.yaml
```

**NOTE**: For lab add `-e @./group_vars/env/lab/all.yaml \`

## Before Moving to Next Steps

Ensure you get the folder and template names for the new cluster you just built, and update the variables in the cluster specific all.yaml files

```sh
cd install-dir
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

#### Install Certs First

```sh
ansible-playbook -i inventory \
-e @./group_vars/certs/cluster-[cluster]-certs.yaml \
-e "cluster=[cluster]" \
--vault-password-file=../resources/vault-password.txt \
certs_deployment_playbook.yaml
```

Wait for all initial nodes to be ready before moving on.
May need to login to the api with kubeadmin since system:admin will be logged out when masters are restarted

#### Finalize Cluster

After the certificates have been applied, and all nodes are back in ready state, log into the api with the kubeadmin username and password provided after the cluster has been initialized.

Update the KUBECONFIG environment variable with the logged in users's tokens:

```sh
export KUBECONFIG=~/.kube/config
```

There will be pauses during the builds, wait until the object just initiated by the playbook has completed before moving on.

The first pause is for the storage nodes, after the machineconfigs have been created. Wait until the storage nodes are up, and vmotion, if needed, the storage nodes before moving on.

Once storage nodes are vmotioned, continue the playbook.

```sh
ansible-playbook -i inventory \
-e @./group_vars/cluster/[cluster]/all.yaml \
-e @./group_vars/env/[env]/default-vault.yaml \
-e @./group_vars/env/[env]/all.yaml \
-e @./group_vars/quay.yaml \
--vault-password-file=../resources/vault-password.txt \
--tags ocp \
finalize_cluster.yaml
```

#### Building Quay Cluster

```sh
ansible-playbook -i inventory \
-e @./group_vars/cluster/[quay_cluster]/all.yaml \
-e @./group_vars/env/[env]]/default-vault.yaml \
-e @./group_vars/env/[env]/all.yaml \
--vault-password-file=../resources/vault-password.txt \
--tags quay \
finalize_cluster.yaml
```

#### Install ArgoCD Projects

Wait 5-10 minutes then run the ArgoCD Projects playbook: [Create ArgoCD Projects](#create-argocd-projects)

## Day 2: Post Initialization Instructions (Individual Playbooks)

Each section below contains the commands for installing the individual plays to finalize a cluster.
You will need to replace the [cluster] defined in each with the cluster you are building (for instance: dev).

### Installing api and wildcard certs

```sh
ansible-playbook -i inventory \
-e @./group_vars/certs/cluster-[cluster]-certs.yaml \
-e "cluster=[cluster]" \
certs_deployment_playbook.yaml \
--vault-password-file=../resources/vault-password.txt
```

### Include Chrony Machineconfigs

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
chrony_playbook.yaml

### Create MachineConfigPool for Infra Nodes

```sh
ansible-playbook -i inventory mcp_playbook.yaml
```

### Installing machinesets

 When running this command you will be prompted for the type of machinesets to be created.
 Defaults to worker, but to build infra machinesets, type: `infra`

```sh
ansible-playbook -i inventory \
-e @./group_vars/env/[env]/default-vault.yaml \
-e @./group_vars/cluster/[cluster]/all.yaml \
-e @./group_vars/env/[env]/all.yaml \
-e "role_assigned=storage" \
--vault-password-file=../resources/vault-password.txt \
machinesets_playbook.yaml
```

### Enable LDAP

```sh
ansible-playbook -i inventory \
-e @./group_vars/env/[env]/default-vault.yaml \
-e @./group_vars/env/[env]/all.yaml \
--vault-password-file=../resources/vault-password.txt \
ldap_playbook.yaml
```

### Enable LDAP Group Sync and role bindings

```sh
ansible-playbook -i inventory \
-e @./group_vars/env/[env]/default-vault.yaml \
-e @./group_vars/cluster/[cluster]/all.yaml \
--vault-password-file=../resources/vault-password.txt \
ldap_groupsync_playbook.yaml
```

### Setting up Infra Taints and Tolerations

**Note**: Make sure all infra nodes are up and provisioned before running this step.

```sh
ansible-playbook -i inventory \
-e @./group_vars/cluster/[cluster]/all.yaml \
nodeconfig_playbook.yaml
```

### Configure OperatorHub to Pull from Disconnected Quay

```sh
ansible-playbook -i inventory \
-e @./group_vars/quay-sync.yaml \
-e "cluster=[cluster]" \
--vault-password-file=../resources/vault-password.txt \
operators_config_playbook.yaml
```

### Configure Update Services to pull from Disconnected Quay

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
update_svs_playbook.yaml
```

### Install an Operator

Pass the variables, depending on the operator, to the playbook.
Variables to pass are:

- Operator Name
- Install Mode (SingleNamespace or AllNamespaces or OwnNamespaces)
- Namespace
- Channel

The above information can be gathered by grepping the information from the packagemanifest.
Defaults to SingleNamespace. If you put AllNamespaces the results will show installed in all namespaces.

To get the required variables grep from the packagemanifest. See next step.

#### Example of grep for Operator Information

This should be done for the following information:

- Install Modes (keyword: modes)
- Suggested Namespace(s) (keyword: suggested)
  - If none is provided, then use the default namespace: openshift-operators
- Install Channel: (keyword: channel)

```sh
oc describe packagemanifest openshift-gitops | grep -A10 -i modes
      Install Modes:
        Supported:  false
        Type:       OwnNamespace
        Supported:  false
        Type:       SingleNamespace
        Supported:  false
        Type:       MultiNamespace
        Supported:  true
        Type:       AllNamespaces
```

#### Run playbook

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster_name]" \
-e "operator=[operator_name]" \
-e "install_mode=[install_mode]" \
-e "ns=[namespace]" \
-e "channel=[channel]" \
-e @./group_vars/cluster/[cluster]/all.yaml \
operators_install_playbook.yaml
```

#### Configs for Operators

- Operator: odf-operator
  - install_mode: OwnNamespace
  - ns: openshift-storage
  - channel: stable-4.12
- Operator: local-storage-operator
  - install_mode: OwnNamespace
  - ns: openshift-local-storage
  - channel: stable
- Operator: openshift-gitops-operator
  - install_mode: AllNamespaces
  - ns: openshift-operators
  - channel: latest
- Operator: openshift-pipelines-operator-rh
  - install_mode: AllNamespaces
  - ns: openshift-operators
  - channel: latest
- Operator: cluster-logging
  - install_mode: OwnNamespace
  - ns: openshift-logging
  - channel: stable
- Operator: elasticsearch-operator
  - install_mode: OwnNamespace
  - ns: openshift-operators-redhat
  - channel: stable
- Operator: openshift-cert-manager-operator
  - install_mode: AllNamespaces
  - ns: cert-manager-operator
  - channel: stable-v1

### Toggle Operator Update Mode

Run the following playbook to set all Operators to Manual Update mode

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
operator_toggle_playbook.yaml
```

### Install ArgoCD

Need to update default-vault.yaml (ansible-vault edit) with the ArgoCD Service Account password, using the username as the key.

```sh
ansible-playbook -i inventory \
-e @./group_vars/cluster/[cluster]/all.yaml \
-e @./group_vars/env/[environment]/all.yaml \
-e @./group_vars/env/[environment]/default-vault.yaml \
--vault-password-file=../resources/vault-password.txt \
install_argocd_playbook.yaml
```

### Install Bitnami

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
bitnami_playbook.yaml
```

### Deploy Network Policy Template

```sh
ansible-playbook -i inventory \
-e @./group_vars/cluster/[cluster]/all.yaml \
network_policy_playbook.yaml
```

### Create ArgoCD Projects

```sh
ansible-playbook -i inventory \
-e @./group_vars/cluster/[cluster]/all.yaml \
net_policy_argo_playbook.yaml
```

### Configure Image Registry

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
image_registry_playbook.yaml
```

### Move Monitoring Pods to Infra Nodes

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
monitoring_config_playbook.yaml
```

## Update Pull Secret

Use vault to update pull-secret.yaml file under /group_vars. Apply below script to apply change on clusters.

```sh
ansible-playbook -i inventory \
-e cluster=[cluster] \
-e @./group_vars/pull-secret.yaml \
update_pullsecret_playbook.yaml \
--vault-password-file=../resources/vault-password.txt
```

## Backup of Bitnami Sealed Secret

1. Create a new git branch for backup
2. Run playbook: sealed_secrets_backup_playbook.yaml
3. Run Scott's script to push backed up secret to git repository.

```sh
ansible-playbook -i inventory \
-e "cluster=[cluster]" \
sealed_secrets_backup_playbook.yaml
```
