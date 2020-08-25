# Creating a Multiple Cluster Demo Environment



## Dependencies

* `kubectl`
* `helm`
* `ytt` from [k14s](https://k14s.io)

If you're on a Mac you can use the `brew bundle` to install these dependencies.

## Intructions

1. Create a new cluster group in TMC.
   ```
   tmc clustergroup create -t default --name ${CLUSTER_GROUP} 
   ```

2. Create your three clusters using TMC's cluster provisioning.
   ```
   tmc cluster create -t default --name ${TOOLS_CLUSTER} --group ${CLUSTER_GROUP} \
      --account-name ${TMC_AWS_CREDENTIAL_NAME} --region ${AWS_REGION} --ssh-key-name ${AWS_SSH_KEY_NAME}
   tmc cluster create -t default --name ${PREPROD_CLUSTER} --group ${CLUSTER_GROUP} \
      --account-name ${TMC_AWS_CREDENTIAL_NAME} --region ${AWS_REGION} --ssh-key-name ${AWS_SSH_KEY_NAME}
   tmc cluster create -t default --name ${PRODUCTION_CLUSTER} --group ${CLUSTER_GROUP} \
      --account-name ${TMC_AWS_CREDENTIAL_NAME} --region ${AWS_REGION} --ssh-key-name ${AWS_SSH_KEY_NAME}
   ```

3. Download the kubeconfigs for the clusters to use for the rest of the steps.
   ```
   tmc cluster provisionedcluster kubeconfig get-admin ${TOOLS_CLUSTER} > secrets/${TOOLS_CLUSTER}.kubeconfig
   tmc cluster provisionedcluster kubeconfig get-admin ${PREPROD_CLUSTER} > secrets/${PREPROD_CLUSTER}.kubeconfig
   tmc cluster provisionedcluster kubeconfig get-admin ${PRODUCTION_CLUSTER} > secrets/${PRODUCTION_CLUSTER}.kubeconfig
   export KUBECONFIG=$(pwd)/secrets/${TOOLS_CLUSTER}.kubeconfig:$(pwd)/secrets/${PREPROD_CLUSTER}.kubeconfig:$(pwd)/secrets/${PRODUCTION_CLUSTER}.kubeconfig

   # contexts are named with the cluster UID instead of the cluster name, JQ to the rescue
   export TOOLS_CONTEXT=kubernetes-admin@$(tmc cluster get --output json ${TOOLS_CLUSTER} | jq -r '.objectMeta.uid | split(":")[1] | ascii_downcase')
   export PREPROD_CONTEXT=kubernetes-admin@$(tmc cluster get --output json ${PREPROD_CLUSTER} | jq -r '.objectMeta.uid | split(":")[1] | ascii_downcase')
   export PRODUCTION_CONTEXT=kubernetes-admin@$(tmc cluster get --output json ${PRODUCTION_CLUSTER} | jq -r '.objectMeta.uid | split(":")[1] | ascii_downcase')
   ```

4. Use Contour as an ingress controller. Strictly speaking we only need to do this for the tools cluster, but 
   you may want to do it for all of the clusters in case you want to show the running application as part of 
   demonstrating that it's running. If you do add it to all clusters, make sure to keep the namespace in the 
   tools workspace.
   ```
   tmc workspace create -t default --name ${TOOLS_WORKSPACE}
   tmc cluster namespace create -t default --name projectcontour --cluster-name ${TOOLS_CLUSTER} --workspace-name ${TOOLS_WORKSPACE}

   # enable a pod security policy for envoy before installing contour
   kubectl apply --context ${TOOLS_CONTEXT} -f config/contour/psp.yml
   kubectl apply --context ${TOOLS_CONTEXT} -f https://projectcontour.io/quickstart/contour.yaml
   ```

4. Install cert-manager into your cluster using Helm. Like the previous step, the demo as 
   written only requires that we do this for the tools cluster but you may want to do it
   across all three clusters if you're showing other things. You still want to use
   the tools workspace for the `cert-manager` namespace when creating it in the other
   clusters.

   ```
   tmc cluster namespace create -t default --name cert-manager --cluster-name ${TOOLS_CLUSTER} --workspace-name ${TOOLS_WORKSPACE}
   helm install --kube-context ${TOOLS_CONTEXT} -n cert-manager great-sunfish jetstack/cert-manager -f values/cert-manager.yml
   ```

5. Install a Let's Encrypt issuer so your registry will have a legitimate
   certificate (it's easier that way).  Again, only needed for the tools cluster
   if you're not expanding on the script.

   ```
   ytt -f config/letsencrypt --data-value email=${EMAIL} | kubectl apply --context ${TOOLS_CONTEXT} -f -
   ```

## Next Steps

After creating the cluster, you need to [prepare TMC](./prepare-tmc.md) 
and then [install and configure Harbor](./harbor-setup.md).
