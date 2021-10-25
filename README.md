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

#### My Finished Script:
```
ibmcloud ks cluster create vpc-gen2 --flavor b3c.4x16 --zone ams03 --workers 3 --name nigel-iks-cluster
```

#### Work in-pogress:
```
ibmcloud ks cluster create vpc-classic --flavor b3c.4x16 --subnet-id 02c7-eef06d9a-650e-4d37-ba61-82cf598ae6af --vpc-id r010-850fb416-7cf7-4381-947e-881c3971ef79 --zone eu-de-1,eu-de-2,eu-de-3 --workers 3 --name nigel-icleks-cluster
```

<img width="1269" alt="Screenshot 2021-10-25 at 20 38 21" src="https://user-images.githubusercontent.com/82048393/138759272-ab4bff54-ee96-440d-bc4e-4a292620a451.png">



## Upgrading to Calico Enterprise

After creating the cluster, assume we are installing for 50+ nodes (this will install Typha): <br/>
https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-more-than-50-nodes 

```
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yaml
```

```
kubectl apply -f calico.yaml
```

#### Setup paste option in VIM

```
:set paste
```

This will be helpful when copying over YAML manifests regularly


#### ScorageClass:
The Storage Classes table displays defined storage classes available to control data sets and objects within a cluster: <br/>
https://www.ibm.com/docs/en/ts7700-virtual-tape/5.0?topic=constructs-storage-classes

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tigera-elasticsearch
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

#### Persistent Volume:
A PV is a virtual storage instance that is added as a volume to the cluster: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-kube_concepts

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-data-tigera-secure-es-7f5dee596fef130f-0
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /var/tigera/elastic-data/1
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: tigera-elasticsearch
```  
#### Load Balancer:
To expose the manager using an IBM load balancer, create the following service: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-loadbalancer

```
kind: Service
apiVersion: v1
metadata:
  name: tigera-manager-external
  namespace: tigera-manager
spec:
  type: LoadBalancer
  selector:
    k8s-app: tigera-manager
  externalTrafficPolicy: Local
  ports:
  - port: 9443
    targetPort: 9443
    protocol: TCP
```

#### Test access to your apps
Create a node port to test your apps: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-nodeport

#### Review other options to expose apps
For production use cases, review the other options to expose your app, such as with load balancers or Ingress: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-cs_network_planning
