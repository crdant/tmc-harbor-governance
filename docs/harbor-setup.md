# Install and Configure Harbor

1. Create a secrets file `secrets/harbor.yml` using the following template:

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
