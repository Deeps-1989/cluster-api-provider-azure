# MachinePools
- **Feature status:** Experimental
- **Feature gate:** MachinePool=true

> In Cluster API (CAPI) v1alpha2, users can create MachineDeployment, MachineSet or Machine custom
> resources. When you create a MachineDeployment or MachineSet, Cluster API components react and
> eventually Machine resources are created. Cluster API's current architecture mandates that a
> Machine maps to a single machine (virtual or bare metal) with the provider being responsible for
> the management of the underlying machine's infrastructure.

> Nearly all infrastructure providers have a way for their users to manage a group of machines
> (virtual or bare metal) as a single entity. Each infrastructure provider offers their own unique
> features, but nearly all are concerned with managing availability, health, and configuration updates.

> A MachinePool is similar to a MachineDeployment in that they both define
> configuration and policy for how a set of machines are managed. They Both define a common
> configuration, number of desired machine replicas, and policy for update. Both types also combine
> information from Kubernetes as well as the underlying provider infrastructure to give a view of
> the overall health of the machines in the set.

> MachinePool diverges from MachineDeployment in that the MachineDeployment controller uses
> MachineSets to achieve the aforementioned desired number of machines and to orchestrate updates
> to the Machines in the managed set, while MachinePool delegates the responsibility of these
> concerns to an infrastructure provider specific resource such as AWS Auto Scale Groups, GCP
> Managed Instance Groups, and Azure Virtual Machine Scale Sets.

> MachinePool is optional and doesn't replace the need for MachineSet/Machine since not every
> infrastructure provider will have an abstraction for managing multiple machines (i.e. bare metal).
> Users may always opt to choose MachineSet/Machine when they don't see additional value in
> MachinePool for their use case.

*Source: [MachinePool API Proposal](https://github.com/kubernetes-sigs/cluster-api/blob/bf51a2502f9007b531f6a9a2c1a4eae1586fb8ca/docs/proposals/20190919-machinepool-api.md)*

## AzureMachinePool
Cluster API Provider Azure (CAPZ) has experimental support for `MachinePool` though the infrastructure
type `AzureMachinePool`. An `AzureMachinePool` corresponds to an [Azure Virtual Machine Scale Set](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview),
which provides the cloud provider specific resource for orchestrating a group of Virtual Machines.

⚠️ Cloud provider for Azure does not currently support clusters in mixed mode (both vmss and vmas node pools), so it is not supported to have both `AzureMachinePools` and `AzureMachines` in the same workload cluster.

### Using `clusterctl` to deploy
To deploy a MachinePool / AzureMachinePool via `clusterctl config` there's a [flavor](https://cluster-api.sigs.k8s.io/clusterctl/commands/config-cluster.html#flavors)
for that.

Make sure to set up your Azure environment as described [here](https://cluster-api.sigs.k8s.io/user/quick-start.html).

```shell
clusterctl config cluster my-cluster --kubernetes-version v1.18.6 --flavor machinepool > my-cluster.yaml
```

The template used for this [flavor](https://cluster-api.sigs.k8s.io/clusterctl/commands/config-cluster.html#flavors)
is located [here](../../templates/cluster-template-machinepool.yaml).

### Example MachinePool, AzureMachinePool and KubeadmConfig Resources
Below is an example of the resources needed to create a pool of Virtual Machines orchestrated with
a Virtual Machine Scale Set.
```yaml
---
apiVersion: exp.cluster.x-k8s.io/v1alpha3
kind: MachinePool
metadata:
  name: capz-mp-0
spec:
  clusterName: capz
  replicas: 2
  template:
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: KubeadmConfig
          name: capz-mp-0
      clusterName: capz
      infrastructureRef:
        apiVersion: exp.infrastructure.cluster.x-k8s.io/v1alpha3
        kind: AzureMachinePool
        name: capz-mp-0
      version: v1.18.6
---
apiVersion: exp.infrastructure.cluster.x-k8s.io/v1alpha3
kind: AzureMachinePool
metadata:
  name: capz-mp-0
spec:
  location: westus2
  template:
    osDisk:
      diskSizeGB: 30
      managedDisk:
        storageAccountType: Premium_LRS
      osType: Linux
    sshPublicKey: ${YOUR_SSH_PUB_KEY}
    vmSize: Standard_D2s_v3
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: KubeadmConfig
metadata:
  name: capz-mp-0
spec:
  files:
  - content: |
      {
        "cloud": "AzurePublicCloud",
        "tenantId": "tenantID",
        "subscriptionId": "subscriptionID",
        "aadClientId": "clientID",
        "aadClientSecret": "secret",
        "resourceGroup": "capz",
        "securityGroupName": "capz-node-nsg",
        "location": "westus2",
        "vmType": "vmss",
        "vnetName": "capz-vnet",
        "vnetResourceGroup": "capz",
        "subnetName": "capz-node-subnet",
        "routeTableName": "capz-node-routetable",
        "loadBalancerSku": "standard",
        "maximumLoadBalancerRuleCount": 250,
        "useManagedIdentityExtension": false,
        "useInstanceMetadata": true
      }
    owner: root:root
    path: /etc/kubernetes/azure.json
    permissions: "0644"
  joinConfiguration:
    nodeRegistration:
      kubeletExtraArgs:
        cloud-config: /etc/kubernetes/azure.json
        cloud-provider: azure
      name: '{{ ds.meta_data["local_hostname"] }}'
  useExperimentalRetryJoin: true
```
