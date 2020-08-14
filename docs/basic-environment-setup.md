# Creating a Basic Demo Environment

The simplest way to set up the demo environment is to use a 
single Kubernetes cluster that TMC creates for you. Following
these steps should have you ready to go in about half an hour.

## Assumptions

1. Your cluster is accessible from the Internet.
2. The registry is for lab and demonstration purposes only.

## Dependencies

* `kubectl`
* `helm`
* `ytt` from [k14s](https://k14s.io)

If you're on a Mac you can use the `brew bundle` to install these dependencies.

## Intructions

1. Create a new cluster group in TMC.
   ```
   ytt -f config/group --data-value name=$CLUSTER_GROUP
   tmc clustergroup create -f work/clustergroup.yaml
   ```
   _N.B._: You have to use the extension `yaml` here rather than `yml` or `tmc` rejects the file

2. Create a new cluster with TMC.
   ```
   ytt -f config/cluster --data-value name=$CLUSTER_NAME --data-value cluster_group=$CLUSTER_GROUP \
      --data-value account_credential=$TMC_AWS_CREDENTIAL_NAME --data-value aws.ssh_key_name=$AWS_SSH_KEY_NAME > work/cluster.yaml
   tmc cluster create -f work/cluster.yaml
   ```
   _N.B._: You have to use the extension `yaml` here rather than `yml` or `tmc` rejects the file

3. Download the kubeconfig for the cluster and use it for the rest of the setup.
   ```
   tmc cluster provisionedcluster kubeconfig get-admin $CLUSTER_NAME > secrets/${CLUSTER_NAME}.kubeconfig
   export KUBECONFIG=$(pwd)/secrets/${CLUSTER_NAME}.kubeconfig
   ```

6. Set up TMC workspaces for governing namespaces used for tool, development, staging, and production workloads.
   ```
   tmc workspace create -t default --name $TOOLS_WORKSPACE
   tmc workspace create -t default --name $DEVELOPMENT_WORKSPACE
   tmc workspace create -t default --name $STAGING_WORKSPACE
   tmc workspace create -t default --name $PRODUCTION_WORKSPACE
   ```

   also create notional workspaces in each workspace

   ```
   tmc cluster namespace create -t default --name development --cluster-name $CLUSTER_NAME --workspace $DEVELOPMENT_WORKSPACE
   tmc cluster namespace create -t default --name crdant --cluster-name $CLUSTER_NAME --workspace $DEVELOPMENT_WORKSPACE
   tmc cluster namespace create -t default --name test --cluster-name $CLUSTER_NAME --workspace $STAGING_WORKSPACE
   tmc cluster namespace create -t default --name staging --cluster-name $CLUSTER_NAME --workspace $STAGING_WORKSPACE
   tmc cluster namespace create -t default --name production --cluster-name $CLUSTER_NAME --workspace $PRODUCTION_WORKSPACE
   ```

4. Use Contour as an ingress controller.
   ```
   tmc cluster namespace create -t default --name contour --cluster-name $CLUSTER_NAME --workspace $TOOLS_WORKSPACE

   # enable a pod security policy for envoy before installing contour
   kubectl apply -f config/contour/psp.yml
   kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
   ```

4. Install cert-manager into your cluster using Helm

   ```
   tmc cluster namespace create -t default --name cert-manager --cluster-name $CLUSTER_NAME --workspace $TOOLS_WORKSPACE
   helm install -n cert-manager great-sunfish jetstack/cert-manager -f values/cert-manager.yml
   ```

5. Install a Let's Encrypt issuer so your registry will have a legitimate
   certificate (it's easier that way). 

   ```
   ytt -f config/letsencrypt --data-value email=$EMAIL | kubectl apply -f -
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
   tmc cluster namespace create -t default --name registry --cluster-name $CLUSTER_NAME --workspace $TOOLS_WORKSPACE
   tmc cluster namespace create -f work/namespace-registry.yaml

   ytt -f config/harbor -f values/harbor.yml --data-value subdomain=$SUBDOMAIN --ignore-unknown-comments > work/harbor.yml 
   helm install -n registry feasible-macaque bitnami/harbor -f secrets/harbor.yml -f work/harbor.yml
   ```
 
9. Prepare Harbor by creating a project and configuring a replication
   endpoint for the Google Container Registry.

   ```
   export HARBOR_PASSWORD=$(kubectl get -n registry secret feasible-macaque-harbor-core-envvars -o jsonpath='{.data.HARBOR_ADMIN_PASSWORD}' | base64 -d)
   curl --user "admin:${HARBOR_PASSWORD}" -X POST https://registry.${SUBDOMAIN}/api/v2.0/projects -H "Content-type: application/json" --data @config/project/project.json
   curl --user "admin:${HARBOR_PASSWORD}" -X POST https://registry.${SUBDOMAIN}/api/v2.0/registries -H "Content-type: application/json" --data @config/replication/gcr.json
   ```

9. Create image registry policies for production and staging workspaces

   ```
   tmc workspace image-policy create -t default-allow-registry --workspace-name $PRODUCTION_WORKSPACE \
     --name private-registry --registry-domains registry.$SUBDOMAIN
   tmc workspace image-policy create -t default-allow-registry --workspace-name $STAGING_WORKSPACE \
     --name trusted-registries --registry-domains registry.$SUBDOMAIN,registry.pivotal.io,gcr.io
   ```

