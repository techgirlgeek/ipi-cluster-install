# Kustomize Configurations

## Summary

This readme document has been edited from the original to reflect how it is being used here.

The original repository, with many other examples, can be found at: <https://github.com/redhat-cop/gitops-catalog/tree/main/oauth>

Parts of the original repository have been cloned and customized to fit this installation, under the ansible/kustomize-dir

## Contents

- [Kustomize Configurations](#kustomize-configurations)
  - [Summary](#summary)
  - [Contents](#contents)
  - [API Certs](#api-certs)
  - [Configuring Identity Providers/OAuth/LDAP](#configuring-identity-providersoauthldap)
    - [Usage](#usage)
    - [LDAP Example](#ldap-example)
  - [LDAP Group Sync Cronjob](#ldap-group-sync-cronjob)
  - [LDAP Role Bindings](#ldap-role-bindings)

## API Certs

Currently we are only using kustomize for applying api certs.

- Create a folder for the cluster you are working on in kustomize-dir/cert-patches/openshift-api-certifcate/overlays/
  - Copy an existing cluster directory to the new one for an example
- Under the directory created above update the api-cert-patch.yaml with the FQDN of the api

## Configuring Identity Providers/OAuth/LDAP

Do not use the `base` directory directly, as you will need to patch the `oauth` and add files to generate secrets.

The current *overlays* in use is for the LDAP provider only:

- [ldap](oauth/overlays/ldap)

### Usage

The LDAP identity provider configured to run in the clusters.

The ansible/ldap-playbook.yml has been configured to apply the Kustomization. To see the configurations that will be applied you can run the following:

```sh
oc apply -k oauth/overlays/ldap/<cluster>
```

As part of a different overlay in your own GitOps repo:

```sh
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - github.com/redhat-cop/gitops-catalog/oauth/overlays/<provider>?ref=main
```

### LDAP Example

The LDAP Playbook deploys the `secret/ldap-secret` and `configmap/ca-config-map` before applying the Kustomization to the cluster.

Example overlay from another repo:

kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - github.com/redhat-cop/gitops-catalog/oauth/overlays/ldap?ref=main

generatorOptions:
  disableNameSuffixHash: true

patchesJson6902:
  - target:
      group: config.openshift.io
      version: v1
      kind: OAuth
      name: cluster
    path: oauth-ldap-patch.yaml
```

oauth-ldap-patch.yaml

```yaml
- op: replace
  path: /spec/identityProviders/0/ldap/url
  value: "ldap://ldap.example.com/ou=users,dc=acme,dc=com?uid"
- op: replace
  path: /spec/identityProviders/0/ldap/bindDN
  value: "ou=users,dc=examplecorp,dc=com"
```

## LDAP Group Sync Cronjob

This code adds the cronjob needed to sync user groups between LDAP and groups in your OpenShift Cluster.

Code can be found at [ldapcron](ldapcron/overlays).

Update items in the oauth-ldap-patch.yaml file, if they change. If production uses a different connection string, or any settings, recommend adding directories (like those found in the api certs overlays directory) for each cluster type, ie: dev, prod, etc.

## LDAP Role Bindings

This section creates [role bindings](groups-role-bindings/base/cluster-ocadmin-group-rolebinding.yaml) between LDAP Groups and OCP Groups. The Groups will be created after the LDAP Cronjob has run and grabbed the groups from LDAP. Users will be added to OCP the first time they successfully log into the cluster.

In the patch file

- Group from LDAP that should be mapped to OCP:
  subjects:
- The role you want to bind to is listed under:
  roleRef:
    name: is the role you want to bind to
