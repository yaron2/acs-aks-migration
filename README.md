# ACS / ACS-Engine to AKS Migration Guide

 A guide for the migration of ACS (Azure Container Service, Unmanaged) / ACS-Engine workloads to AKS (Azure Container Service, Managed).

 *This is a living document, currently up to date as of May 1st, 2018.*

## Purpose and motivation

The benefits of a fully managed Kubernetes service are kicking in as Kubernetes gains more ground in the public cloud space.
So far, users have been using ACS and ACS-Engine (The Open Source template builder for Docker enabled clusters on Azure) for their production workloads.
These offerings are unmanaged, and once deployed, the responsibility for managing the cluster rests on the user.<br>
This is still the recommended way to go for production workloads until AKS is Generally Available, but when the time comes to move from ACS or ACS-Engine to AKS, it's important to have a clear understanding of the differences between the offerings and be able to come up with a migration path.

## Scope

This document will focus on a high level feature comparison between ACS and ACS-Engine to AKS and then deep dive into specific important subjects to take into consideration such as networking, stateful migrations, autoscale and more.

Out of scope are:

* AKS Roadmap References
* Live, realtime migration of stateful workloads
* AKS Support and SLA differences from ACS

## Overview

AKS is still in preview, and does not currently support all the features of ACS and ACS-Engine.
As ACS-Engine is the building block for both ACS and AKS, you can expect certain features from ACS-Engine to move to AKS with time.

Before migrating to AKS, please familiarize yourself with the following [AKS FAQ] (https://docs.microsoft.com/en-us/azure/aks/faq).

### Feature comparison overview

Feature | ACS-Engine        | ACS           | AKS   |
--------| :------------------:|:-------------:| :-----:|
Managed Disks        | V          | V (In preview, selected regions) |  V |
Unmanaged Disks |   V  |  V | X
Managed Masters | X | X | V
Automated Upgrades | X | X | V
Pay for | Masters + Nodes | Masters + Nodes | Nodes 
Network Policies        | Calico, Cilium         | X      |   X |
Network Plugins       | Kubenet, Azure CNI     | Kubenet      |    Kubenet |
Custom VNET | V | V (In preview, selected regions) | X
Private Cluster (No Public IPs) | V | X | X
KeyVault Encryption (etcd) | V | X | X
RBAC | V | X | X
Multiple Node Pools | V | X | X
Windows Nodes | V | V | X
Operating Systems | Ubuntu, CoreOS | Ubuntu | Ubuntu
Managed Identity | V | X | X
VMSS Support | V (1.10+) | X | X
Autoscale | V (1.10+) | X | X


### Stateless migration

Migrating stateless workloads that do not rely on persistent data are relatively simple to migrate.<br>
Once you made sure your target cluster supports all features used by your deployments, you can procceed with moving the Kubernetes resources.

If you have a CI/CD pipeline in place such as Jenkins or VSTS, you can update your kubeconfig in your build environment to point to the new cluster and deploy your resources to the new cluster.

Example:
Code Repository --> CI/CD Pipeline --> Container Registry --> CI/CD Pipeline --> AKS

For detailed information on using Jenkins with AKS, see [here] (https://azure.microsoft.com/en-us/solutions/architecture/container-cicd-using-jenkins-and-kubernetes-on-azure-container-service/)


### Stateful migration

Migrating stateful workloads is trickier as they require Azure infrastructure operations such as moving the data between disks as well as the corresponding Kubernetes resources.

AKS supports Managed Disks only, so any migration of Unmanaged Disks to AKS must involve a conversion of the VHDs to Managed Disks.

There is an [azure kube cli extension] (https://github.com/yaron2/azure-kube-cli) that helps automate all the steps below.

#### Non Managed Disk to Managed Disk (Same Region)

1) Create a snapshot from Blob URI
2) Create a Managed Disk from the blob snapshot in the AKS resource group that has the MC_ prefix


#### Non Managed Disk to Managed Disk (Cross Region)

1) Create a snapshot from Blob URI
2) Create a temp storage account in the target AKS resource group that has the MC_ prefix
3) Copy the blob snapshot between cross region storage accounts
4) Create a Managed Disk from the blob snapshot (temp storage account) in the AKS resource group that has the MC_ prefix
5) (Optional) - Clean up temp storage account


#### Managed Disk to Managed Disk (Same Region)

1) Create a snapshot from Managed Disk
2) Create a Managed Disk from the blob snapshot in the AKS resource group that has the MC_ prefix


#### Managed Disk to Managed Disk (Cross Region)

1) Create a snapshot from Managed Disk
2) Create a temp storage account in the target AKS resource group that has the MC_ prefix
3) Copy the blob snapshot between cross region storage accounts
4) Create a Managed Disk from the blob snapshot (temp storage account) in the AKS resource group that has the MC_ prefix
5) (Optional) - Clean up temp storage account


#### Point Persistent Volumes to existing Managed Disk

Now that you've copied your data to newly created Azure Managed Disks, it's time to create the Kubernetes Persistent Volumes and configure them to use the existing disks.

Here's an example of a Persistent Volume:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {name}
spec:
  capacity:
    storage: 10Gi
  storageClassName: default
  azureDisk:
    kind: Managed
    diskName: {name}
    diskURI: /subscriptions/{subscription-id}/resourceGroups/{aks-controlled-resource-group-name}/providers/Microsoft.Compute/disks/{disk-name}
    fsType: ext4
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    name: {name}-pvc
    namespace: default
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {name}-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: default
  ```


### IPs and traffic redirection

It is advisable to use DNS records that point to your IP addresses, and once the resource migration to the new cluster id done, you can simply point the DNS record to the newly create Load Balancer IP addresses of the new cluster.

Example flow:

1) Migrate resources from old cluster to new cluster
2) Go to your DNS registar and point your DNS addresses to the corresponding IP addresses in the new cluster
3) Use ```kubectl drain $NODENAME``` on the nodes of your old cluster to allow for a graceful termination of pods on a node and then marking it as unschedulable

### VNET support

AKS doesn't currently support a custom VNET deployment.
AKS will create one VNET for the agnets with a 10.0.0.0/8 address range, and one subnet with a 10.240.0.0/16 range.

If you need to peer your vnet with other vnets or establish a S2S VPN, you need to take this default range into consideration as network overlap may occur.

### Network policies and plugins

AKS does not currently network policies such as Calico and Cilium.
If you require that pods in your cluster can only be accessed by authorized pods, that might be an issue.

Note that you can also use the NetworkPolicy resource to control traffic between pods in a cluster.

AKS does not currently support the Azure CNI, so if have a scenario that requires pods to communicate to to VMs in peered VNETs or on-premises networks over ExpressRoute and VPNs and vice versa and you are not able to mirror the routes, this may be a point to re-consider if AKS currently fits your scenario.

For more information on container networking, see [this] (https://docs.microsoft.com/en-us/azure/virtual-network/container-networking) link.

### Autoscale

ACS and ACS-Engine both have custom implementations of auto-scaling, such as the [acs-engine-autoscaler] (https://github.com/wbuchwalter/Kubernetes-acs-engine-autoscaler) and [ACS autoscaler] (https://github.com/kim0/Kubernetes-acs-autoscaler)

Starting from Kubernetes 1.10, ACS-Engine has [autoscale support] (https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/azure/README.md) both VMSS and VMs.

AKS does have support in cluster-autoscaler as of now, but there's an [issue] (https://github.com/kubernetes/autoscaler/issues/753) you can track here for progress.

A standalone implementation for autoscaling AKS can be found [here] (https://github.com/yaron2/aks-autoscaler)
