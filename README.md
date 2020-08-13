# Image Governance with Harbor and Tanzu Mission Control

This repository contains the code and configuration needed to 
demonstrate container image governance with [Tanzu Mission
Control](https://tanzu.vmware.com/mission-control) and 
[Harbor](https://goharbor.io).

## Creating the Environment

This demonstration assumes that you have a Harbor registry 
available and a cluster attached to TMC. If you don't have
a registry set up, you can set one up using the following 
steps:

1. Create a new cluster group in TMC.
   ```
   ytt -f config/group --data-value name=$CLUSTER_GROUP
   tmc clustergroup create -f work/clustergroup.yaml
   ```
   _N.B._: You have to use the extension `yaml` here rather than `yml` or `tmc` rejects the file

2. Create a new cluster with TMC.
   ```
   ytt -f config/cluster --data-value name=$CLUSTER_NAME --data-value cluster_group=$CLUSTER_GROUP --data-value account_credential=$TMC_AWS_CREDENTIAL_NAME --data-value aws.ssh_key_name=$AWS_SSH_KEY_NAME > work/cluster.yaml
   tmc cluster create -f work/cluster.yaml
   ```
   _N.B._: You have to use the extension `yaml` here rather than `yml` or `tmc` rejects the file

3. Download the kubeconfig for the cluster and use it for the rest of the setup.
   ```
   tmc cluster provisionedcluster kubeconfig get-admin $CLUSTER_NAME > secrets/${CLUSTER_NAME}.kubeconfig
   export KUBECONFIG=$(pwd)/secrets/${CLUSTER_NAME}.kubeconfig
   ```

4. Install cert-manager into your cluster using Helm

   ```
   kubectl create namespace cert-manager
   helm install -n cert-manager great-sunfish jetstack/cert-manager -f values/cert-manager.yml
   ```

5. Install a Let's Encrypt issuer so your registry will have a legitimate
   certificate (it's easier that way). 

   ```
   ytt -f config/letsencrypt --data-value email=$EMAIL | kubectl apply -f -
   ```

6. Set up TMC workspaces for governing namespaces used for tool, development, staging, and production workloads.
   ```
   tmc workspace create -f config/workspaces/tools.yaml
   tmc workspace create -f config/workspaces/development.yaml
   tmc workspace create -f config/workspaces/staging.yaml
   tmc workspace create -f config/workspaces/production.yaml
   ```

7. Create a secrets file `secrets/harbor.yml` using the following template:

   ```
   harborAdminPassword:

   core:
     secretKey:
     secret:
   postgresql:
     password:
   ``` 

8. Install Harbor with Helm 

   ```
   ytt -f config/harbor -f values/harbor.yml --data-value subdomain=$SUBDOMAIN --ignore-unknown-comments > work/harbor.yml 
   kubectl create namespace registry
   helm install -n registry feasible-macaque bitnami/harbor -f secrets/harbor.yml -f work/harbor.yml
   ```

## Assumptions

1. You want to use a Let's Encrypt certificate.
2. Your cluster is accessible from the Internet.
3. The registry is for lab and demonstration purposes only.

## Dependencies

* `kubectl`
* `helm`
* `ytt` from [k14s](https://k14s.io)

If you're on a Mac you can use the `brew bundle` to install these dependencies.
