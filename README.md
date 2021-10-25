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

<img width="1187" alt="Screenshot 2021-10-25 at 21 32 58" src="https://user-images.githubusercontent.com/82048393/138766647-0aae772f-5379-421e-86bc-8aa420adc584.png">

Install the Tigera ```Custom Resources Defintions``` (CRD's):
```
kubectl create -f https://docs.tigera.io/manifests/custom-resources.yaml
```

<img width="1187" alt="Screenshot 2021-10-25 at 21 35 21" src="https://user-images.githubusercontent.com/82048393/138766927-f5396f1f-718b-4b1c-be41-826eaf2ce3f3.png">

You can now monitor progress with the following command:
```
watch kubectl get tigerastatus
```

<img width="524" alt="Screenshot 2021-10-25 at 21 37 32" src="https://user-images.githubusercontent.com/82048393/138767271-e0797b62-f2e9-46c5-b794-126587860bbe.png">

Wait until the ```apiserver``` shows a status of ```Available```, then proceed to the next section:

<img width="524" alt="Screenshot 2021-10-25 at 21 39 43" src="https://user-images.githubusercontent.com/82048393/138767616-2d9a39e1-d592-45af-9904-298a9605390d.png">

#### Install Calico Enterprise License:
```
kubectl create -f license.yaml
```

<img width="528" alt="Screenshot 2021-10-25 at 21 43 09" src="https://user-images.githubusercontent.com/82048393/138768021-e7c64d00-0df0-4f4a-9056-790d38af22a7.png">

Check all pods are still creating:

```
kubectl get pods -A
```

<img width="807" alt="Screenshot 2021-10-25 at 21 44 57" src="https://user-images.githubusercontent.com/82048393/138768253-4f2c2c77-4b83-4af9-9ab4-7f064ab67eba.png">

If not an issue with pods creating, check the persistent volume claim status:

```
kubectl get pvc -A
```

<img width="1082" alt="Screenshot 2021-10-25 at 21 47 31" src="https://user-images.githubusercontent.com/82048393/138768519-fd55f60f-6d40-4518-8af2-f928c13c787e.png">



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

<img width="1467" alt="Screenshot 2021-10-25 at 21 51 43" src="https://user-images.githubusercontent.com/82048393/138769121-1621c28e-6e0a-4a92-914f-863f2d6ef724.png">


Confirm the new pods are getting created after the persistent volume changes:
``` 
kubectl get pods -A
```

<img width="1467" alt="Screenshot 2021-10-25 at 21 53 09" src="https://user-images.githubusercontent.com/82048393/138769331-85f1f4ca-0077-4d52-8ea7-f3c3badc506e.png">

This process could take 3/5 mins to complete in IKS. <br/>
The outcome should look something like the below:

<img width="501" alt="Screenshot 2021-10-25 at 21 57 35" src="https://user-images.githubusercontent.com/82048393/138769861-a790b927-dd64-440c-95a6-35be80374aea.png">


<img width="839" alt="Screenshot 2021-10-25 at 21 58 52" src="https://user-images.githubusercontent.com/82048393/138769957-0d913c6e-ea9d-45be-9651-bd524b4530a6.png">

When all components show a status of ```Available```, continue to the next section.

#### Secure Calico Enterprise with network policy:
Install the following network policies to secure Calico Enterprise component communications.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies.yaml
```

<img width="677" alt="Screenshot 2021-10-25 at 22 03 30" src="https://user-images.githubusercontent.com/82048393/138770579-26ecc1ed-4080-4684-89cd-e9de9b09def7.png">


Please note that the below command only returns policies in the default ```tier```:
```
kubectl get globalnetworkpolicy
```

<img width="677" alt="Screenshot 2021-10-25 at 22 03 30" src="https://user-images.githubusercontent.com/82048393/138770714-9578593c-d0ff-4a53-a7ec-da5d1d9ab2c5.png">



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

<img width="811" alt="Screenshot 2021-10-25 at 22 16 09" src="https://user-images.githubusercontent.com/82048393/138772182-802a4291-1228-473b-a498-746e40cb90d2.png">


#### Test Access to your Apps
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


#### Authentication Quickstart:
Get started quickly with our default token authentication to log in to Calico Enterprise Manager UI and Kibana. <br/>
```
kubectl create sa nigel -n default
```

Give the service account permissions to access the Calico Enterprise Manager UI, and a Calico Enterprise cluster role:
```
kubectl create clusterrolebinding nigel-access --clusterrole tigera-network-admin --serviceaccount default:nigel
```

Get Base64 encoded token for SA account - ```nigel```:
```
kubectl get secret $(kubectl get serviceaccount nigel -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```
Connect to Kibana with the elastic username. Use the following command to decode the password:
```
kubectl -n tigera-elasticsearch get secret tigera-secure-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' && echo
```
Once logged in, you can configure users and their privileges from the settings page.


<img width="1448" alt="Screenshot 2021-10-25 at 22 25 34" src="https://user-images.githubusercontent.com/82048393/138773268-c81858f0-b5e8-4867-95d5-e6bd03a4ed88.png">




