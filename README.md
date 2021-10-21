# Upgrade IKS Cluster from Calico OSS to Calico Enterprise

**Goal:** Create an IKS clusters compatible for Calico Enterprise

After you've [installed the CLIs](https://cloud.ibm.com/docs/containers?topic=containers-cs_cluster_tutorial#cs_cluster_tutorial_lesson1) for running Kubernetes commands and managing Docker images, you can create your first cluster! <br/>
Be sure to specify the cluster configurations that fit your needs.


## Prerequisite Checks

General prereq checks are listed here: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-cs_cluster_tutorial#cs_cluster_tutorial_lesson1 <br/>
<br/>
Install IBM Cloud CLI or use the cloud shell <br/>
https://cloud.ibm.com/docs/cli?topic=cli-getting-started


## Setting environmental variables

Finding the zones for the classic provider
```
ibmcloud ks zone ls --provider classic
```

Finding the flavours in the Amsterdam zone

```
ibmcloud ks flavors --zone ams03
```

Using ```b3c.4x16``` because it has 4 cores with 16GB of memory:

```
ibmcloud ks flavors --zone ams03
```

Get the level of permissions assigned to the user
```
ibmcloud ks infra-permissions get
```

These commands help you gather the information that you need to create a virtual server instance: <br/>
https://cloud.ibm.com/vpc-ext/overview


## Miscellaneous

```
ibmcloud is vpcs
```
```
ibmcloud is subnets
```

## Installing the cluster in Cloud Shell

Default Script:
```
ibmcloud ks cluster create --location dal10 --public-vlan <public_vlan_id> --private-vlan <private_vlan_id> --machine-type b2c.4x16 --workers 3 --name <cluster_name>
```

My Finished Script:
```
ibmcloud ks cluster create vpc-gen2 --flavor b3c.4x16 --zone ams03 --workers 3 --name nigel-iks-cluster
```
