# Prepare TMC for the Demo

1. Set up TMC workspaces for governing namespaces used for development, 
   staging, and production workloads. We set up the workspace for our
   tools when configuring the tools cluster.
   
   ```
   tmc workspace create -t default --name ${DEVELOPMENT_WORKSPACE}
   tmc workspace create -t default --name ${STAGING_WORKSPACE}
   tmc workspace create -t default --name ${PRODUCTION_WORKSPACE}
   ```

2. Create notional namespaces for each environment in the development lifecycle. TMC
   TMC will create the namespaces in the appropriate cluster. If you're working with 
   only one cluster, then use the same cluster as the development, test, and 
   production clusters.

   ```
   tmc cluster namespace create -t default --name development --cluster-name ${PREPROD_CLUSTER} --workspace-name ${DEVELOPMENT_WORKSPACE}
   tmc cluster namespace create -t default --name crdant --cluster-name ${PREPROD_CLUSTER} --workspace-name ${DEVELOPMENT_WORKSPACE}
   tmc cluster namespace create -t default --name test --cluster-name ${PREPROD_CLUSTER} --workspace-name ${STAGING_WORKSPACE}
   tmc cluster namespace create -t default --name staging --cluster-name ${PREPROD_CLUSTER} --workspace-name ${STAGING_WORKSPACE}
   tmc cluster namespace create -t default --name production --cluster-name ${PRODUCTION_CLUSTER} --workspace-name ${PRODUCTION_WORKSPACE}
   ```

3. Create image registry policies for production and staging workspaces

   ```
   tmc workspace image-policy create -t default-allow-registry --workspace-name $PRODUCTION_WORKSPACE \
     --name private-registry --registry-domains registry.${SUBDOMAIN}
   tmc workspace image-policy create -t default-allow-registry --workspace-name $STAGING_WORKSPACE \
     --name trusted-registries --registry-domains registry.${SUBDOMAIN},registry.pivotal.io,gcr.io
   ```

