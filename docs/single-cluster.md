# Creating a Single Cluster Demo Environment

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
   tmc clustergroup create -t default --name ${CLUSTER_GROUP} 
   ```

2. Create a new cluster with TMC.
   ```
   tmc cluster create -t default --name ${CLUSTER_NAME} --group ${CLUSTER_GROUP} --account-name ${TMC_AWS_CREDENTIAL_NAME} --ssh-key-name ${AWS_SSH_KEY_NAME}
   ```

3. Download the kubeconfig for the cluster and use it for the rest of the setup.
   ```
   tmc cluster provisionedcluster kubeconfig get-admin $CLUSTER_NAME > secrets/${CLUSTER_NAME}.kubeconfig
   export KUBECONFIG=$(pwd)/secrets/${CLUSTER_NAME}.kubeconfig
   ```

4. Use Contour as an ingress controller.
   ```
   tmc workspace create -t default --name ${TOOLS_WORKSPACE}
   tmc cluster namespace create -t default --name projectcontour --cluster-name $CLUSTER_NAME --workspace $TOOLS_WORKSPACE

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

## Next Steps

After creating the cluster, you need to [prepare TMC](./prepare-tmc.md) 
and then [install and configure Harbor](./harbor-setup.md).
