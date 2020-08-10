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

1. Install cert-manager into your cluster using Helm

   ```helm install great-sunfish jetpack/cert-manager -f values/cert-manager.yml```

2. Install a Let's Encrypt issuer so you're registry will have a legitimate
   certificate (it's easier that way). 

   ```
   ytt -f config/letsencrypt --data-value email=$EMAIL | kubectl -f - i
   ```

2. Create a secrets file `secrets/harbor.yml` using the following template:

   ```
   harborAdminPassword:

   core:
     secretKey:
     secret:
   postgresql:
     password:
   ``` 

3. Install Harbor with Helm 

   ```
   ytt -f config/harbor -f values/harbor.yml > work/harbor.yml --data-value subdomain=$SUBDOMAIN
   helm install feasible-macaque bitnami/harbor -f secrets/harbor.yml -f work/harbor.yml
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
