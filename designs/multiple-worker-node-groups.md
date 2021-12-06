# Multiple worker node groups

## Introduction

**Problem:** Currently users of eks-anywhere can only create one worker node group by specifying the configuration details in a cluster spec.

This limits the ability of a user to create worker nodes with different configurations and formulate application deployment strategy based on the configurations of the worker nodes.

Also, this creates a problem while tainting a node. Unless, eks-anywhere clusters support multiple worker node groups, worker nodes can not be tainted with `NoExecute` effect, since nodes in a node group have the same configuration, setting a taint with `NoExecute` effect would essentially make the worker nodes unusable for eks-anywhere specific deployments, as the general approach for eks-anywhere is to not add tolerations on deployments.


### Tenets

***Simple:*** Specifying different node group configurations in the cluster spec should be simple and readable.

### Goals and Objectives

As a Kubernetes administrator I want to:

* Add multiple worker node group configurations in the `workerNodeGroupConfigurations` array in cluster specs.
* Have the ability to point each worker node group configuration to a different machine configuration.
* Have the ability to point multiple worker node groups to same machine config.
* Specify separate node counts and taints information for each node group.

### Statement of Scope

**In scope**

* Providing users the ability to add multiple worker node groups configuration in the cluster spec and bootstrap kubernetes clusters with multiple worker node groups.

## Overview of Solution

With this feature, a user can create a cluster config file with multiple worker node group configurations. Upon running eks-anywhere cli, these info will be fetched and be added in the CAPI specification file. In the CAPI spec, configuration details will be appended for each worker node groups one after another. Examples of each of these two files are added in the next section.
 
## Solution Details

With this feature, worker node specific parts of the cluster spec file will look like below.

```
  workerNodeGroupConfigurations:
  - count: 3
    machineGroupRef:
      kind: VSphereMachineConfig
      name: eksa-test-1
      Taints:
      - key: Key2
        value: value2
        effect: PreferNoSchedule
  - count: 3
    machineGroupRef:
      kind: VSphereMachineConfig
      name: eksa-test-2
      Taints:
      - key: Key3
        value: value3
        effect: PreferNoSchedule
status: {}
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereMachineConfig
metadata:
  creationTimestamp: null
  name: eksa-test-1
spec:
...
status: {}

---
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereMachineConfig
metadata:
  creationTimestamp: null
  name: eksa-test-2
spec:
...
status: {}

---
```

Once it is processed through cli, the generated CAPI spec file should have the worker nodes specific configurations like below.

```
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: KubeadmConfigTemplate
metadata:
  name: eksa-test-1-md-0
  namespace: eksa-system
spec:
  template:
    spec:
      joinConfiguration:
        pause:
          imageRepository: public.ecr.aws/eks-distro/kubernetes/pause
          imageTag: v1.20.7-eks-1-20-8
        bottlerocketBootstrap:
          imageRepository: public.ecr.aws/l0g8r8j6/bottlerocket-bootstrap
          imageTag: v1-20-8-eks-a-v0.0.0-dev-build.579
        nodeRegistration:
          criSocket: /var/run/containerd/containerd.sock
          kubeletExtraArgs:
            cloud-provider: external
            tls-cipher-suites: Something
          name: '{{ ds.meta_data.hostname }}'
	  ...
---
apiVersion: cluster.x-k8s.io/v1alpha3
kind: MachineDeployment
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: eksa-test
  name: eksa-test-1-md-0
  namespace: eksa-system
spec:
  clusterName: eksa-test
  replicas: 3
  selector:
    matchLabels: {}
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: eksa-test
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: KubeadmConfigTemplate
          name: eksa-test-1-md-0
      clusterName: eksa-test
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
        kind: VSphereMachineTemplate
        name: eksa-test-worker-node-template-1638469395669
      version: v1.20.7-eks-1-20-8
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: VSphereMachineTemplate
metadata:
  name: eksa-test-worker-node-template-1638469395669
  namespace: eksa-system
spec:
  template:
    spec:
    ...
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: KubeadmConfigTemplate
metadata:
  name: eksa-test-2-md-0
  namespace: eksa-system
spec:
  template:
    spec:
      joinConfiguration:
        pause:
          imageRepository: public.ecr.aws/eks-distro/kubernetes/pause
          imageTag: v1.20.7-eks-1-20-8
        bottlerocketBootstrap:
          imageRepository: public.ecr.aws/l0g8r8j6/bottlerocket-bootstrap
          imageTag: v1-20-8-eks-a-v0.0.0-dev-build.579
        nodeRegistration:
          criSocket: /var/run/containerd/containerd.sock
          kubeletExtraArgs:
            cloud-provider: external
            tls-cipher-suites: Something
          name: '{{ ds.meta_data.hostname }}'
	  ...
---
apiVersion: cluster.x-k8s.io/v1alpha3
kind: MachineDeployment
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: eksa-test
  name: eksa-test-2-md-0
  namespace: eksa-system
spec:
  clusterName: eksa-test
  replicas: 3
  selector:
    matchLabels: {}
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: eksa-test
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: KubeadmConfigTemplate
          name: eksa-test-2-md-0
      clusterName: eksa-test
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
        kind: VSphereMachineTemplate
        name: test
      version: v1.20.7-eks-1-20-8
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: VSphereMachineTemplate
metadata:
  name: test
  namespace: eksa-system
spec:
  template:
    spec:
    ...
---
```

For each worker node groups, CAPI spec file will continue to have the following 3 kind fields.

* KubeadmConfigTemplate
* Machime deployment
* VSphereMachineTemplate

For each group, we will append these three fields corresponding to that group in the CAPI spec.

Right now, the cli assumes that there will be only one group and it treats worker node group configuration array as a collection of only one element. As a result, the controller just refers to the first element of this array in different places of the code. So we need to do the same operations in loops, which includes CAPI spec creation, cluster spec validation etc. Once a CAPI spec is created with this approach, the workload cluster will be created with multiple worker nodes.

Also, it needs to be made sure that at the least one of the worker node groups does not have `NoExecute` taint. This validation will be done at the preflight validation stage.

The examples in this design are for vsphere provider. But the same strategy applies for other providers as well.

## Testing

To make sure that the implementation of this feature is correct, we need to add unit tests for each providers to validate the correctness of generated CAPI specs.

Also, we need to add e2e tests for each providers to test the following scenarios.

* Cluster creation with one worker node group
* Cluster creation with multiple worker node groups

## Conclusion

Current implementation of eks-anywhere cli implements an array of worker node group configurations and assumes that the array has only one element. With this design we can enhance the scope the current implementation to make sure it can handle multiple elements in that array. This will help us achieve our goal to support multiple worker node groups.