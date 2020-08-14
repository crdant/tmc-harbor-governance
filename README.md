# Image Governance with Harbor and Tanzu Mission Control

This repository contains the code and configuration needed to 
demonstrate container image governance with [Tanzu Mission
Control](https://tanzu.vmware.com/mission-control) and 
[Harbor](https://goharbor.io).

## Scenario

The demonstration considers a context in which a team is has
a development process that builds, tests, and deploys software
using multiple environments. Each environment has different
constraints, with development environments able to run any
software and production limited to software that has passed
through a governance process that includes security scanning.

We combine a few TMC and Harbor features to demonstrate a 
layered governance model to enforce these constraints. 

* Image policies in TMC assure that images can only be pulled 
  if they are from a registry trusted for the given environment.
* Replication allows taking an image from another registry 
  and using the trusted private registry to deliver it.
* Security scanning in Harbor assure that even when pulled from
  our trusted private repository, we cannot run an image with 
  known vulnerabilities. 
* The CVE Whitelist feature in Harbor allows teams to make risk
  aware judgements to allow software with known vulnerabilities
  to run.

## Running the demonstration

To run the demonstration, you need one or more Kubernetes
clusters, access to Tanzu Mission Control, and an installed
Harbor registry. You can [setup a basic environment]
(docs/basic-environment-setup.md) using a single cluster in
about 30 minutes. The instructions assume you followed those
steps, but are easy to adapt if you made adjustments.

### Steps

4. Before you start, make sure you have a shell open that 
   has access to the cluster. If you're in the shell used
   for the [basic setup](docs/basic-environment-setup.md)
   you used for the basic setup you should be all set. If
   not, make sure that the `KUBECONFIG` environment is set
   to allow access to the demonstration cluster.
   ```
   export KUBECONFIG=$(pwd)/secrets/${CLUSTER_NAME}.kubconfig
   ```

 1. Log into TMC and walk through the cluster and workspace 
   cluster for the demo. From the [basic setup 
   instructions](docs/basic-environment-setup.md) you'll 
   have a single cluster group containing one cluster along
   with four workgroups.

2. Show the cluster and the namespaces within it.  Point out
   how each namespaces is in a workspaces and the role that 
   workspaces play in policy management. Note that there are
   workspaces representing development, staging, and production
   environments since they have different constaints that 
   can be enforced with policy. Not that there's also a tools
   workgroup for tools that are internal to the process like
   the Harbor registry.

3. Follow the Policy menu and speak about image registry 
   policies. Share navigate to the workspaces for the three
   types of environments and show how both staging and 
   production have image registry policies with different
   registries allowed. Note that development is wide open
   to allow for experimentation.

4. Switch to your shell and depoly the [Kubernetes Up and
   Running demo application](https://github.com/kubernetes-up-and-running/kuard).
   I chose this since it's fairly well known and if you 
   deploy the version currently referenced in its README
   then you're deploying an image with a vulnerability.
   ```
   $ kubectl run -n development --restart=Never --image-pull-policy=Always --image=gcr.io/kuar-demo/kuard-amd64:blue kuard
   pod/kuard created
   $ kubectl get pods -n development
   NAME    READY   STATUS    RESTARTS   AGE
   kuard   1/1     Running   0          23h
   ```
   It will deploy and run in the development environment 
   since there is no image registry policy.

5. Deploy again to the `test` namespace, which has an 
   image registry policy.
   ```
   $ kubectl run -n test --restart=Never --image-pull-policy=Always --image=gcr.io/kuar-demo/kuard-amd64:blue kuard
   pod/kuard created
   $ kubectl get pods -n test
   NAME    READY   STATUS    RESTARTS   AGE
   kuard   1/1     Running   0          23h
   ```
   This run will also succeed, since the staging image 
   policy allows you to pull for `gcr.io`. Pop on over to
   TMC and point it out.

6. Now you're going to try to deploy to the production. This will
   fail since the image pull policy doesn't allow images from 
   `gcr.io`.
   ```
   $ kubectl run -n test --restart=Never --image-pull-policy=Always --image=gcr.io/kuar-demo/kuard-amd64:blue kuard
   Error from server (failed to match image policies: spec.containers[0].image: Forbidden: no matching image policy): admission webhook "validation.policies.tmc.cloud.vmware.com" denied the request: failed to match image policies: spec.containers[0].image: Forbidden: no matching image policy
   ```

7. Explain how the organization can use image replication
   to take third-party images into the trusted registry and
   scan them as part of the process. Manully set up the 
   replication for the `kuar-demo/kuard-amd64` image and
   kick it off. Make sure you specify the `blue` tag to
   speed up the replication and assure that it gets scanned
   promptly.

8. Navigate to the `kuard` project and show how the repository
   was replicated. Filter with the tag `blue` and point out the 
   failed scan. Take note of the number of the "High" severity
   CVE (CVE-2019-14697).

8. Once the replication is completed, try deploying again,
   this time with the image from the local registry.
   ```
   $ kubectl run -n production  --restart=Never --image=registry.tools.aws.crdant.io/kuard/kuar-demo/kuard-amd64:blue kuard
   pod/kuard created
   $ kubectl get pods -n production
   NAME    READY   STATUS             RESTARTS   AGE
   kuard   0/1     ImagePullBackOff   0          9s
   ```
   You should also run `kubectl describe -n production pod kuard`
   to show the error message.


10. Jump back over to Harbor and click on the project configuration
    to show how it's configured to "Prevent vulnerable images from
    running." Show how you can adjust the level, and set it to
    "High".  Explain that there is still a "High" vulnerability
    in the image, but we can use the whitelist to allow it for
    this project. Add the CVE (CVE-2019-14697) to the whitelist.

11. Back in the shell, watch Kubernetes continue to try to run the
    pod until it next tries to pull the image with the whitelist in
    place. This will show the pod get to `Running` state when the
    image is pulled.
    ```
    
    NAME    READY   STATUS         RESTARTS   AGE
    kuard   0/1     ErrImagePull   0          39s
    kuard   0/1     ImagePullBackOff   0          46s
    kuard   1/1     Running            0          49s
    ```
