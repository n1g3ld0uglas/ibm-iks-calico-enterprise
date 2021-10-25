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

Wait for all worker nodes to be in a Ready status for Calico changes to be accepted:

<img width="1269" alt="Screenshot 2021-10-25 at 20 54 29" src="https://user-images.githubusercontent.com/82048393/138761486-7a49b94a-c1d4-4419-91d6-5bbf50d43cf5.png">


## Upgrading to Calico Enterprise:

Confirm the Calico pods are running as expected:
```
kubectl get pods -A
```
<img width="1313" alt="Screenshot 2021-10-25 at 21 00 26" src="https://user-images.githubusercontent.com/82048393/138762410-61b9739d-be33-4502-80df-148538d419bd.png">

After creating the cluster, assume we are installing for 50+ nodes (this will install Typha): <br/>
https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-more-than-50-nodes 

```
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yaml
```

```
kubectl apply -f calico.yaml
```

<img width="888" alt="Screenshot 2021-10-25 at 21 03 48" src="https://user-images.githubusercontent.com/82048393/138762722-28ac7df1-e0fa-4ca0-bf30-9d5b1073fe97.png">

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

<img width="1103" alt="Screenshot 2021-10-25 at 21 10 13" src="https://user-images.githubusercontent.com/82048393/138763590-6847207b-a169-46da-acff-9dac600f3dec.png">

Install the ```Tigera Operator``` and custom resource definitions:

```
kubectl create -f https://docs.tigera.io/manifests/tigera-operator.yaml
```

<img width="1560" alt="Screenshot 2021-10-25 at 21 20 44" src="https://user-images.githubusercontent.com/82048393/138765079-d33c4271-8ec5-4cce-94a0-412eb46815f1.png">

Install the ```Prometheus Operator``` and related custom resource definitions:

```
kubectl create -f https://docs.tigera.io/manifests/tigera-prometheus-operator.yaml
```

The ```Prometheus Operator``` will be used to deploy Prometheus server and Alertmanager to monitor Calico Enterprise metrics:

<img width="745" alt="Screenshot 2021-10-25 at 21 23 40" src="https://user-images.githubusercontent.com/82048393/138765368-cc9a207a-5c34-42bb-a15c-9aea437f2e05.png">

#### Install your Pull Secret:

```
kubectl create secret generic tigera-pull-secret --from-file=.dockerconfigjson=config.json --type=kubernetes.io/dockerconfigjson -n tigera-operator
```

If pulling images directly from ```quay.io/tigera```, you will likely want to use the credentials provided to you by your Tigera support representative.




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

#### Test access to your Apps
Create a node port to test your apps: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-nodeport

```
apiVersion: v1
kind: Service
metadata:
  name: <my-nodeport-service>
  labels:
    <my-label-key>: <my-label-value>
spec:
  selector:
    <my-selector-key>: <my-selector-value>
  type: NodePort
  ports:
   - port: <8081>
     # nodePort: <31514>
```

#### Review other options to Expose Apps:
For production use cases, review the other options to expose your app, such as with load balancers or Ingress: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-cs_network_planning <br/>
<br/>
How Kubernetes forwards public network traffic through NodePort, LoadBalancer, and Ingress services in IBM Cloud Kubernetes Service:

![cs_network_planning_ov-01](https://user-images.githubusercontent.com/82048393/138760195-80b093bc-4018-4788-b1d5-514f05b6e171.png)



