# Upgrade IKS Cluster from Calico OSS to Calico Enterprise

Quick note: It's better to ```UPGRADE``` from OSS Calico to CaliEnt - and not to ```INSTALL``` CaliEnt on top of Calico OSS: <br/>
https://docs.tigera.io/v3.10/maintenance/upgrade-to-tsee


#### Create cluster via IBM cloud shell
```
ibmcloud ks cluster create vpc-gen2 --flavor b3c.4x16 --zone ams03 --workers 3 --name nigel-iks-cluster
```

#### Create cluster via the web builder

To meet the Calico Enterprise minimum prerequisite, I gave my ```worker pool``` 16GB of memory and 4 vCPU's:
![Screenshot 2021-12-07 at 17 13 39](https://user-images.githubusercontent.com/82048393/145075710-e3c9e1fb-9712-4359-a03a-5058951053be.png)

To reduce costs, I also recommend ```disabling integrations``` that are unwanted (such as logging and monitoring):
![Screenshot 2021-12-07 at 17 14 04](https://user-images.githubusercontent.com/82048393/145075814-80838984-c673-4809-9427-f8830f3986be.png)

Wait for all worker nodes to be in a ```Ready``` status for Calico changes to be accepted:
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

Run the below command and wait for the ```calico-typha``` pods to be created into the ```kube-system``` namespace:

```
kubectl get pods -A -w
```

This could take between 2/3 mins to complete

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


Confirm the new persistent volume claim is in a ```BOUND``` status:
``` 
kubectl get pvc -A -w
```

A persistent volume claim (PVC) is a request for storage, which is met by binding the PVC to a persistent volume (PV). <br/>
This PVC provides an abstraction layer to the underlying storage. <br/>
https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingpersistentvolumeclaim.htm

Install the ```Tigera Operator``` and custom resource definitions:

```
kubectl apply -f https://docs.tigera.io/manifests/tigera-operator.yaml
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
kubectl apply -f https://docs.tigera.io/manifests/custom-resources.yaml
```

Check the pod status again. The ```calico-node``` pods should be re-built in the ```calico-system``` namespace:

```
kubectl get pods -A -w
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
kubectl get pods -A -w
```

<img width="807" alt="Screenshot 2021-10-25 at 21 44 57" src="https://user-images.githubusercontent.com/82048393/138768253-4f2c2c77-4b83-4af9-9ab4-7f064ab67eba.png">

<img width="1467" alt="Screenshot 2021-10-25 at 21 53 09" src="https://user-images.githubusercontent.com/82048393/138769331-85f1f4ca-0077-4d52-8ea7-f3c3badc506e.png">

This process could take 3/5 mins to complete in IKS. <br/>
The outcome should look something like the below:

<img width="501" alt="Screenshot 2021-10-25 at 21 57 35" src="https://user-images.githubusercontent.com/82048393/138769861-a790b927-dd64-440c-95a6-35be80374aea.png">

Double-check that pods aren't ```crashing``` or reverting back to a ```PodInitializing``` status

<img width="839" alt="Screenshot 2021-10-25 at 21 58 52" src="https://user-images.githubusercontent.com/82048393/138769957-0d913c6e-ea9d-45be-9651-bd524b4530a6.png">

When all components show a status of ```Available```, continue to the next section.



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

<img width="1127" alt="Screenshot 2021-10-25 at 22 31 37" src="https://user-images.githubusercontent.com/82048393/138774098-b19c261b-e460-417f-ae10-922dc50089fa.png">

#### Calico Enterprise Login Page

This should be the expected login page for Calico Enterprise:
<img width="709" alt="Screenshot 2021-11-15 at 15 37 44" src="https://user-images.githubusercontent.com/82048393/141810361-7896a001-3fae-4028-9d4e-eab315a9d236.png">



#### Review other options to Expose Apps:
For production use cases, review the other options to expose your app, such as with load balancers or Ingress: <br/>
https://cloud.ibm.com/docs/containers?topic=containers-cs_network_planning <br/>
<br/>
How Kubernetes forwards public network traffic through NodePort, LoadBalancer, and Ingress services in IBM Cloud Kubernetes Service:

![cs_network_planning_ov-01](https://user-images.githubusercontent.com/82048393/138760195-80b093bc-4018-4788-b1d5-514f05b6e171.png)


## Authentication Quickstart (SUPER USER):
Get started quickly with our default token authentication to log in to Calico Enterprise Manager UI and Kibana. <br/>
```
kubectl create sa damien -n default
```

Give the service account permissions to access the Calico Enterprise Manager UI, and a Calico Enterprise cluster role:
```
kubectl create clusterrolebinding damien-access --clusterrole tigera-network-admin --serviceaccount default:damien
```

Get Base64 encoded token for SA account - ```damien```:
```
kubectl get secret $(kubectl get serviceaccount damien -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```
Connect to Kibana with the elastic username. Use the following command to decode the password:
```
kubectl -n tigera-elasticsearch get secret tigera-secure-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' && echo
```
Once logged in, you can configure users and their privileges from the settings page.


<img width="1448" alt="Screenshot 2021-10-25 at 22 25 34" src="https://user-images.githubusercontent.com/82048393/138773268-c81858f0-b5e8-4867-95d5-e6bd03a4ed88.png">

### Create a lesser-privleged service account - (READ-ONLY USER)

The roles and bindings in this file provide a minimum starting point for setting up RBAC
```
wget https://docs.tigera.io/getting-started/kubernetes/installation/hosted/cnx/demo-manifests/min-ui-user-rbac.yaml
```

Run this command to replace with the name or email of the user you are permitting. In this case it's ```ons```
```
sed -i -e 's/<USER>/ons/g' min-ui-user-rbac.yaml
```

Use the following command to install the bindings
```
kubectl apply -f min-ui-user-rbac.yaml
```

Create a 2nd Service account ```ons``` with limited permissions
```
kubectl create serviceaccount ons
```

Create an Admin user with limited UI access to the Calico Enterprise Manager:

```
kubectl create clusterrolebinding ons-access --clusterrole tigera-ui-user --serviceaccount default:ons
```

Get the token from the service account ```ons```
```
kubectl get secret $(kubectl get serviceaccount ons -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```

## Secure Calico Enterprise with network policy
Install the following network policies to secure Calico Enterprise component communications.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies.yaml
```

## Create a management cluster
To control managed clusters from your central management plane, you must ensure it is reachable for connections. <br/>
The simplest way to get started (but not for production scenarios), is to configure a NodePort service to expose the management cluster. <br/> 
Note that the service must live within the tigera-manager namespace. <br/>
https://docs.tigera.io/multicluster/mcm/create-a-management-cluster

```
apiVersion: v1
kind: Service
metadata:
  name: tigera-manager-mcm
  namespace: tigera-manager
spec:
  ports:
  - nodePort: 30449
    port: 9449
    protocol: TCP
    targetPort: 9449
  selector:
    k8s-app: tigera-manager
  type: NodePort
```




Export the service port number, and the public IP or host of the management cluster.
```
export MANAGEMENT_CLUSTER_ADDR=<your-management-cluster-addr>:9443
```

Apply the ManagementCluster Custom Resource (CR):
```
apiVersion: operator.tigera.io/v1
kind: ManagementCluster
metadata:
  name: tigera-secure
spec:
  address: $MANAGEMENT_CLUSTER_ADDR
```

#### Create an admin user and verify management cluster connection

Create an admin user called, mcm-user in the default namespace with full permissions, by applying the following commands.

```
kubectl create sa mcm-user
kubectl create clusterrolebinding mcm-user-admin --serviceaccount=default:mcm-user --clusterrole=tigera-network-admin
kubectl get secret $(kubectl get serviceaccount mcm-user -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo

```

## Create a managed cluster

Follow these steps in the cluster you intend to use as the managed cluster. <br/>
https://docs.tigera.io/multicluster/mcm/create-a-managed-cluster

#### Download the Tigera custom resources (For Modification)

```
curl -O -L https://docs.tigera.io/manifests/custom-resources.yaml
```

Remove the ```Manager``` custom resource from the manifest file.
```
apiVersion: operator.tigera.io/v1
kind: Manager
metadata:
  name: tigera-secure
spec:
  # Authentication configuration for accessing the Tigera manager.
  # Default is to use token-based authentication.
  auth:
    type: Token
```

Remove the ```LogStorage``` custom resource from the manifest file.

```
apiVersion: operator.tigera.io/v1
kind: LogStorage
metadata:
  name: tigera-secure
spec:
  nodes:
    count: 1
```

Now apply the modified manifest:

```
kubectl create -f ./custom-resources.yaml
```

You can now monitor progress with the following command:
```
watch kubectl get tigerastatus
```

Wait until the ```apiserver``` shows a status of Available, then proceed to the next section.

#### Create the connection manifest for your managed cluster:

Because you will eventually have several managed clusters, choose a name that can be easily recognized in a list of managed clusters.

```
export MANAGED_CLUSTER=my-managed-cluster
```

Add a managed cluster and save the manifest containing a ManagementClusterConnection and a Secret:
```
kubectl -o jsonpath="{.spec.installationManifest}" > $MANAGED_CLUSTER.yaml create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: ManagedCluster
metadata:
  name: $MANAGED_CLUSTER
EOF
```

Verify that the ```managementClusterAddr``` in the manifest is correct.


#### Apply the connection manifest to your managed cluster:

Apply the manifest that you modified in the step, Add a managed cluster to the management cluster.
```
kubectl apply -f $MANAGED_CLUSTER.yaml
```

Monitor progress with the following command:
```
watch kubectl get tigerastatus
```
Wait until the ```management-cluster-connection``` and ```tigera-compliance``` show a status of Available.

#### Provide permissions to view the managed cluster:
```
kubectl create clusterrolebinding mcm-user-admin --serviceaccount=default:mcm-user --clusterrole=tigera-network-admin
```

## Introduce a test application into your environment

If your cluster does not have applications, you can use the following ```storefront``` application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```
<img width="698" alt="Screenshot 2021-11-15 at 16 25 28" src="https://user-images.githubusercontent.com/82048393/141817656-d9f6fd40-5845-4d97-90cf-6d14435b6559.png">

Confirm that the pods are present in your ```Kubernetes Dashboard``` view:

<img width="1080" alt="Screenshot 2021-11-15 at 17 57 30" src="https://user-images.githubusercontent.com/82048393/141831178-6d3fe2df-feb6-4d4d-a991-d1af06c35588.png">

You can also confirm which pods are running via the Calico Enterprise web user interface:

<img width="1098" alt="Screenshot 2021-11-15 at 18 03 02" src="https://user-images.githubusercontent.com/82048393/141831797-376fc95b-1e7d-4467-a08d-4127dd50309e.png">

#### Confirm labels of your pods

```
kubectl get pods -n storefront --show-labels
```

#### Manual process of testing policies

Create the ```Product``` Tier:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/product.yaml
```  
<img width="813" alt="Screenshot 2021-11-15 at 16 26 44" src="https://user-images.githubusercontent.com/82048393/141817844-ed9f7cde-c21c-480b-a35c-92194fc2cc8d.png">


## Zone-Based Architecture  
Create the ```DMZ``` Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/dmz.yaml
```

#### YAML Output (DMZ)
```
apiVersion: projectcalico.org/v3
kind: StagedNetworkPolicy
metadata:
  name: product.dmz
  namespace: storefront
spec:
  tier: product
  order: 0
  selector: fw-zone == "dmz"
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source:
        selector: ipset == "internal"
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination:
        selector: fw-zone == "trusted"||app == "logging"
    - action: Deny
      source: {}
      destination: {}
  types:
    - Ingress
    - Egress

```

Create the ```Trusted``` Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/trusted.yaml
``` 
Create the ```Restricted``` Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/restricted.yaml
```

#### Confirm all policies are running:
```
kubectl get networkpolicies.p -n storefront -l projectcalico.org/tier=product
```

![Screenshot 2021-11-15 at 17 49 53](https://user-images.githubusercontent.com/82048393/141830156-cdce2117-2722-41ed-a394-8ffa6164fe54.png)

We ultimately need to decide between a zone-based architecture (or allow traffic on a per-pod basis). <br/>
NB: The below screenshot deomstrates the options in 2 separate tiers:

<img width="1061" alt="Screenshot 2021-11-15 at 17 54 58" src="https://user-images.githubusercontent.com/82048393/141830739-255ae682-cf01-465a-a3a7-df3b5fcc4eb0.png">




## Allow Kube-DNS Traffic: 
We need to create the following policy within the ```tigera-security``` tier <br/>
Determine a DNS provider of your cluster (mine is 'coredns' by default)
```
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```    
Allow traffic for Kube-DNS / CoreDNS:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/allow-kubedns.yaml
```


![Screenshot 2021-11-15 at 18 27 38](https://user-images.githubusercontent.com/82048393/141834934-fd7de958-f291-4901-be99-3da595a6712d.png)





## Scaling your IKS Cluster

Get the name of the worker pool that you want to resize:
```
ibmcloud ks worker-pool ls --cluster nigel-iks-ce-test
```

<img width="532" alt="Screenshot 2021-11-11 at 21 58 01" src="https://user-images.githubusercontent.com/82048393/141374904-80743e67-5527-4f92-96b3-cef5fb91dee4.png">




Resize the worker pool by designating the number of worker nodes that you want to deploy in each zone:
```
ibmcloud ks worker-pool resize --cluster nigel-iks-ce-test --worker-pool nigel-worker-pool  --size-per-zone 0
```

<img width="914" alt="Screenshot 2021-11-11 at 21 59 42" src="https://user-images.githubusercontent.com/82048393/141374980-dffd4c11-b88c-4f2a-b767-939dabd446f0.png">

## Pull Images for Private Repo:
Move Calico Enterprise container images to a private registry and configure Calico Enterprise to pull images from it:
<br/>
https://docs.tigera.io/getting-started/private-registry/private-registry-regular

Add tigera-pull-secret into the namespace tigera-internal:
```
kubectl create secret generic tigera-pull-secret --from-file=.dockerconfigjson=<pull-secrets.json> --type=kubernetes.io/dockerconfigjson -n tigera-internal
```

Apply the following manifest to create a namespace and RBAC for the honeypods:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/common.yaml 
```

#### IP Enumeration
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/ip-enum.yaml 
```

#### Nginx Service
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/expose-svc.yaml 
```

Verify the Honeypods are deployed:
```
kubectl get pods -n tigera-internal
```

And verify that global alerts are set for honeypods:
```
kubectl get globalalerts
```

## Calico Deep Packet Inspection
Configuring DPI using Calico Enterprise <br/>
Security teams need to run DPI quickly in response to unusual network traffic in clusters so they can identify potential threats. 

### Introduce a test application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Also, it is critical to run DPI on select workloads (not all) to efficiently make use of cluster resources and minimize the impact of false positives.

### Bring in a Rogue Application
```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml
```

Calico Enterprise provides an easy way to perform DPI using Snort community rules.

### Create DeepPacketInspection resource: 
In this example we will enable DPI on backend pod in storefront namespace:

```
apiVersion: projectcalico.org/v3
kind: DeepPacketInspection
metadata:
  name: database
  namespace: storefront
spec:
  selector: app == "backend"
```

You can disable DPI at any time, selectively configure for namespaces and endpoints, and alerts are generated in the Alerts dashboard in Manager UI. 

### Check that the "tigera-dpi" pods created successfully
It's a deaemonSet so one pod should created in each node:

```
kubectl get pods -n tigera-dpi
```

### Make sure that all pods are in running state
Trigger Snort rule from attacker pod to backend.storefront

```
kubectl exec -it $(kubectl get po -l app=attacker-app -ojsonpath='{.items[0].metadata.name}') -- sh -c "curl http://backend.storefront.svc.cluster.local:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'"
```

### Now, go and check the Alerts page in the UI
You should see a signature triggered alert. <br/>
Once satisfied with the alerts, you can disable Deep Packet Inspection via the below command:
```
kubectl delete DeepPacketInspection database -n storefront 
```

### Hipstershop Reference
```
apiVersion: projectcalico.org/v3
kind: DeepPacketInspection
metadata:
  name: hipstershop-dpi-dmz
  namespace: hipstershop
spec:
  selector: zone == "dmz"
```

## IBM IKS clusters should also connect with Calico Cloud

Prerequisite checks are much the same for Calico Cloud as they are for Calico Enterprise: <br/>
https://docs.calicocloud.io/operations/connect/gke#prerequisites


![Screenshot 2022-01-12 at 09 19 10](https://user-images.githubusercontent.com/82048393/149100496-dcb40f36-2711-4ede-a854-21b0fa48c863.png)

## Anonymization Attacks:  
Create the threat feed for ```KNOWN-MALWARE``` which we can then block with network policy: 
``` 
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/malware-ipfeed.yaml
```

Create the threat feed for ```Tor Bulk Exit``` Nodes: 

``` 
kubectl apply -f https://docs.tigera.io/manifests/threatdef/tor-exit-feed.yaml
```

Additionally, feeds can be checked using following command:

``` 
kubectl get globalthreatfeeds 
```

As you can see from the below example, it's making a pull request from a dynamic feed and labelling it - so we have a static selector for the feed:
```
apiVersion: projectcalico.org/v3
kind: GlobalThreatFeed
metadata:
  name: vpn-ejr
spec:
  pull:
    http:
      url: https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/ejrfeed.txt
  globalNetworkSet:
    labels:
      threatfeed: vpn-ejr
```

Applies to anything that IS NOT listed with the namespace selector = 'acme' 

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/block-feodo.yaml
```
  
## Setup for Honeypods testing

Create the Tigera-Internal namespace and alerts for the honeypod services: <br/>
https://docs.tigera.io/threat/honeypod/honeypods

```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/common.yaml
```

Add tigera-pull-secret into the namespace tigera-internal:
```
kubectl create secret generic tigera-pull-secret --from-file=.dockerconfigjson=<pull-secrets.json> --type=kubernetes.io/dockerconfigjson -n tigera-internal
```

#### Pulling Images for the Honeypods
Expose an empty pod that can only be reached via PodIP; this allows you to see when the attacker is probing the pod network:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/ip-enum.yaml
```

Expose a nginx service that serves a generic page. The pod can be discovered via ClusterIP or DNS lookup. <br/>
An unreachable service tigera-dashboard-internal-service is created to entice the attacker to find and reach, tigera-dashboard-internal-debug:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/expose-svc.yaml 
```

Expose a vulnerable SQL service that contains an empty database with easy access.<br/>
The pod can be discovered via ClusterIP or DNS lookup:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/vuln-svc.yaml 
```

#### Verify honeypods deployment

Verify the deployment - ensure that honeypods are running within the tigera-internal namespace:

```
kubectl get pods -n tigera-internal -o wide
```

And verify that global alerts are set for honeypods:

```
kubectl get globalalerts
```

As an example, to trigger an alert for ```honeypod.ip.enum```, first get the Pod IP for one of the honeypods:
```
kubectl get pod tigera-internal-app-28c85 -n tigera-internal -ojsonpath='{.status.podIP}'
```
Then, run a ```busybox``` container with the command ping on the honeypod IP:
```
kubectl run --restart=Never --image busybox ping-runner -- ping -c1 <honeypod IP>
``` 
If the ICMP request reaches the honeypod, an alert will be generated within 5 minutes.<br/>
<br/>
After you have verified that the honeypods are installed and working, a best practice is to ```remove the pull secret``` from the namespace:
```
kubectl delete secret tigera-pull-secret -n tigera-internal
```

## Enable packet capture on honeypods

The following manifest enables packet capture on default honeypods. <br/>
Be sure to modify the namespace and selector if honeypods are placed elsewhere: <br/>
https://docs.tigera.io/visibility/packetcapture

```
kubectl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: PacketCapture
metadata:
  name: capture-honey
  namespace: tigera-internal
spec:
  selector: all()
EOF
```

#### Add honeypod controller to cluster

Add the honeypod controller to each cluster configured for honeypods using the following command:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/controller.yaml
```
#### Add custom Snort rules

You can add custom Snort rules to the controller using a ConfigMap: <br/>
https://docs.tigera.io/threat/honeypod/honeypod-controller <br/>
<br/>
By default, Calico Enterprise uses the Snort Community Ruleset:
https://www.snort.org/downloads/#rule-downloads <br/>
<br/>
The following manifest provides the method to add individual custom signatures:
```
kubectl create -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: localrule
  namespace: tigera-intrusion-detection
data:
  rules: |
    alert icmp any any -> any any (msg:"ICMP Echo Request"; itype:8; sid:1000000;)
    alert icmp any any -> any any (msg:"ICMP Echo Reply"; itype:0; sid:1000001;)
EOF
```



## Cache Refresh
```
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"dnsLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFileAggregationKindForAllowed":1}}'
```
