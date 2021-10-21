# Upgrade IKS Cluster from Calico OSS to Calico Enterprise
Create an IKS clusters compatible for Calico Enterprise

**Goal:** Create EKS cluster.

After you've [installed the CLIs](https://cloud.ibm.com/docs/containers?topic=containers-cs_cluster_tutorial#cs_cluster_tutorial_lesson1) for running Kubernetes commands and managing Docker images, you can create your first cluster! <br/>
Be sure to specify the cluster configurations that fit your needs.


## Install Options


Default Script
```
ibmcloud ks cluster create --location dal10 --public-vlan <public_vlan_id> --private-vlan <private_vlan_id> --machine-type b2c.4x16 --workers 3 --name <cluster_name>
```
Limitation Script
```
ibmcloud ks cluster create --machine-type b2c.4x16 --workers 3 --name nigel-iks-cluster
```
