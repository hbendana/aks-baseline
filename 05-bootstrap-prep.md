# Prep for cluster bootstrapping

Now that the [hub-spoke network is provisioned](./04-networking.md), the next step in the [AKS baseline reference implementation](./) is preparing what your AKS cluster should be bootstrapped with.

## Expected results

Container registries often have a lifecycle that extends beyond the scope of a single cluster. They can be scoped broadly at organizational or business unit levels, or can be scoped at workload levels, but usually are not directly tied to the lifecycle of any specific cluster instance. For example, you may do blue/green _cluster instance_ deployments, both using the same container registry. Even though clusters came and went, the registry stays intact.

* Azure Container Registry (ACR) is deployed, and exposed as a private endpoint.
* ACR is populated with images your cluster will need as part of its bootstrapping process.
* Log Analytics is deployed and ACR platform logging is configured. This workspace will be used by your cluster as well.

The role of this pre-existing ACR instance is made more prominant when we think about cluster bootstrapping. That is the process that happens after Azure resource deployment of the cluster, but before your first workload lands in the cluster. The cluster will be bootstrapped _immedately and automatically_ after resource deployment, which means you'll need ACR in place to act as your official OCI artifact repository for required images and Helm charts used in that bootstrapping process.

### Method

We'll be bootstrapping this cluster with the Flux GitOps agent as installed as an AKS extension. This specific choice does not imply that Flux, or GitOps in general, is the only approach to bootstrapping. Consider your organizational familiarity and acceptance of tooling like this and decide if cluster bootstrapping should be performed with GitOps or via your deployment pipelines. If you are running a fleet of clusters, a GitOps approach is highly recommended for uniformity and easier governance. When running only a few clusters, GitOps might be seen as "too much" and you might instead opt for integrating that process into one or more deployment pipelines to ensure bootstrapping takes place. No matter which way you go, you'll need your bootstrapping artifacts ready to go before you start your cluster deployment so that you can minimize the time between cluster deployment and bootstrapping. Using the Flux AKS extension allows your cluster to start already bootstrapped and sets you up with a solid management foundation going forward.

## Steps

1. Create the AKS cluster resource group.

   > :book: The app team working on behalf of business unit 0001 (BU001) is looking to create an AKS cluster of the app they are creating (Application ID: 0008). They have worked with the organization's networking team and have been provisioned a spoke network in which to lay their cluster and network-aware external resources into (such as Application Gateway). They took that information and added it to their [`acr-stamp.json`](./acr-stamp.json), [`cluster-stamp.json`](./cluster-stamp.json), and [`azuredeploy.parameters.prod.json`](./azuredeploy.parameters.prod.json) files.
   >
   > They create this resource group to be the parent group for the application.

   ```bash
   export AKS_AKS_BASELINE="rg-bu0001a0008-"$rgNumber
   echo AKS_AKS_BASELINE: $AKS_AKS_BASELINE

   # [This takes less than one minute.]
   az group create --name $AKS_AKS_BASELINE --location eastus2
   ```

1. Get the AKS cluster spoke virtual network resource ID.

   > :book: The app team will be deploying to a spoke virtual network, that was already provisioned by the network team.

   ```bash
   export RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE=$(az deployment group show -g $SPOKES_AKS_BASELINE -n spoke-BU0001A0008 --query properties.outputs.clusterVnetResourceId.value -o tsv)
   ```

1. Deploy the container registry template.

   ```bash
   # [This takes about four minutes.]
   az deployment group create -g $AKS_AKS_BASELINE -f acr-stamp.bicep -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE}
   ```

1. Import cluster management images to your container registry.

   > Public container registries are subject to faults such as outages or request throttling. Interruptions like these can be crippling for a system that needs to pull an image _right now_. To minimize the risks of using public registries, store all applicable container images in a registry that you control, such as the SLA-backed Azure Container Registry.

   ```bash
   # Get your ACR instance name
   export ACR_NAME_AKS_BASELINE=$(az deployment group show -g $AKS_AKS_BASELINE -n acr-stamp --query properties.outputs.containerRegistryName.value -o tsv)

   # Import core image(s) hosted in public container registries to be used during bootstrapping
   az acr import --source docker.io/weaveworks/kured:1.9.0 -n $ACR_NAME_AKS_BASELINE
   ```

   > In this walkthrough, there is only one image that is included in the bootstrapping process. It's included as an reference for this process. Your choice to use Kubernetes Reboot Daemon (Kured) or any other images, including helm charts, as part of your bootstrapping is yours to make.

### Save your work in-progress

```bash
# run the saveenv.sh script at any time to save environment variables created above to aks_baseline.env
./saveenv.sh

# if your terminal session gets reset, you can source the file to reload the environment variables
# source aks_baseline.env
```

### Next step

:arrow_forward: [Deploy the AKS cluster](./06-aks-cluster.md)
