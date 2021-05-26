# aks-calico-enterprise
Deploying a compatible AKS cluster for Calico Enterprise


Create a Azure Resource Group in your region. Simply put, resource groups (RG's) are a collection of assets in logical groups for easy or even automatic provisioning, monitoring, and access control, and for more effective management of their costs:
https://www.otava.com/reference/how-to-use-azure-resource-groups-a-simple-explanation/

```
az group create --name nigelResourceGroup --location northeurope
```
<img width="909" alt="Screenshot 2021-05-26 at 10 18 28" src="https://user-images.githubusercontent.com/82048393/119645850-62bc5600-be16-11eb-92c3-f33b9850b58e.png">

Next we will create the AKS cluster. This has been configured with 3 worker nodes (since managed k8's don't worry about the need for a master node).
The VM type is 'Standard_B2ms' - this way we don't run into any issues with CPU/Memory allocation during the installation process.

Calico Enterprise required an x86-64 processor with at least 2 cores, 8.0GB RAM and 20 GB free disk space.
https://docs.tigera.io/getting-started/bare-metal/requirements#node-requirements

The Standard_B2ms VM provides 2 vCPU's and 8.0GiB of Memory - ideal for our deployment:
https://docs.microsoft.com/en-us/azure/virtual-machines/sizes-b-series-burstable

We are also selecting the 'azure-cni' as our network plugin - this allows us to use transparent networking mode:
https://docs.microsoft.com/en-us/azure/aks/faq#what-is-azure-cni-transparent-mode-vs-bridge-mode

```
az aks create --resource-group nigelResourceGroup --name nigelAKSCluster --node-vm-size Standard_B2ms --node-count 3 --zones 1 2 3 --network-plugin azure
```

<img width="667" alt="Screenshot 2021-05-26 at 10 24 43" src="https://user-images.githubusercontent.com/82048393/119646172-c34b9300-be16-11eb-951e-21b7bd8310d8.png">


Calico CNI is currently not supported in with transparent mode in AKS - this will break the install process is chosen.
Before proceeding, I would merge "nigelAKSCluster" as the current context within /home/nigel/.kube/config

```
az aks get-credentials --resource-group nigelResourceGroup --name nigelAKSCluster
```

I also confirm the nodes are in each of the failire zones specified within my cluster deployment:
```
kubectl describe nodes | grep -e "Name:" -e "failure-domain.beta.kubernetes.io/zone"
```

The goal is to have each node in it's own zone within the same region, in the case where the one zone fails:
```
kubectl get nodes -o custom-columns=NAME:'{.metadata.name}',REGION:'{.metadata.labels.topology\.kubernetes\.io/region}',ZONE:'{metadata.labels.topology\.kubernetes\.io/zone}'
```

If you ever need planned downtime for the cluster, you can easily scale down the node count to 0 - revert back to 3 nodes when needed:
```
az aks scale --resource-group nigelResourceGroup -name nigelAKSCluster -node-count 0
```

Now for Calico Enterprise. We will start by configuring the StorageClass.
StorageClass tells Calico Enterprise to use the GCE Persistent Disks for log storage:

```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/netpolTest/main/sc/afs.yaml
```

Install the Tigera operator and custom resource definitions
```
kubectl create -f https://docs.tigera.io/manifests/tigera-operator.yaml
```

Install the Prometheus operator and related custom resource definitions
```
kubectl create -f https://docs.tigera.io/manifests/tigera-prometheus-operator.yaml
```

Create a config.json file in the current directory with the token provided to you by the team at Tigera:
```
vi config.json
```

Install your pull secret - referencing the same config.json file
```
kubectl create secret generic tigera-pull-secret --type=kubernetes.io/dockerconfigjson -n tigera-operator --from-file=.dockerconfigjson=config.json
```

Install the AKS-specific custom resources
```
kubectl create -f https://docs.tigera.io/manifests/aks/custom-resources.yaml
```

Monitor progress with the following command. This can take a 4-5 mins (usually)
We are waiting on for 'apiserver' and 'calico' to show as 'true' under 'AVAILABLE'
```
watch kubectl get tigerastatus
```

If there is any significant delay in this changed, get a status on all running pods:
```
kubectl get pods -A
```

You can 'describe' - get information from a specific pod within a namespace that is not successfully running:
```
kubectl describe pod tigera-secure-kb-8848584d7-dclwb -n tigera-kibana
```

Create the license file (again, provided by the team at Tigera) so we can apply the update:
```
vi license.yaml
```

In order for the remaining components to work, you must install the license provided to you by Tigera
```
kubectl apply -f license.yaml
```

You can now monitor progress with the following command - this might take a few minutes to complete.
As always, you can check the staus of pod creation by regularly checking - kubectl get pods -A
```
watch kubectl get tigerastatus
```

You will start seeing the new pods with status 'ContainerCreating' after the license is accepted.
Don't be afraid to run kubectl describe command if manager is taking too long to appear:

```
kubectl describe pod tigera-manager-77b7f9d674-8rl27 -n tigera-manager
```

Secure Calico component communications, install this set of network policies
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies-managed.yaml
```

To expose the manager using a load balancer, create the following service
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/netpolTest/main/sc/lb.yaml
```

After creating the service, it may take a few minutes for the load balancer to be created.
You can access Calico Enterprise via 'EXTERNAL-IP:PORT(443)'
```
kubectl get services -n tigera-manager tigera-manager-external
```

First, create a service account in the desired namespace.
A Service Account (SA) must be located in a specific namespace.
```
kubectl create sa nigel -n default
```
Give the service account, Nigel in the default namespace network admin permissions (basically super admin priveleges)
```
kubectl create clusterrolebinding nigel-access --clusterrole tigera-network-admin --serviceaccount default:nigel
```

Get the token from the service account
```
kubectl get secret $(kubectl get serviceaccount nigel -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```

Log into the web UI with the above base64 encoded token.
This is the basic authentication method - however, you can integrate with your idP for more secure authentication.

Since you ran the below 'tigera-policies-managed.yaml' file already, you should have an 'allow-tigera' tier created.
This tier is responsible for ensuring all Calico/Tigera componets work smoothly. It has a higher precendence over all tier down to the 'default' tier.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies-managed.yaml
```

You can access Kibana via your 'EXTERNAL-IP:9443/tigera-kibana/login'
The username is 'elastic' by default, and the password token can be generated via the below command:

```
kubectl -n tigera-elasticsearch get secret tigera-secure-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' && echo
```

CalicoCTL (Calico CLI tool) is not installed with Calico Enterprise by default. 
If this is not installed already, I would recommend installing this utility:

```
curl -o calicoctl -O -L "https://github.com/projectcalico/calicoctl/releases/download/v3.19.0/calicoctl"
```

I prefer to convert this into an executable. 
However, there are other options on how to use calicoctl:
https://docs.projectcalico.org/getting-started/clis/calicoctl/install
```
chmod +x calicoctl
```

In our deployment, we have AKS set to auto allocate IP blocks. This can be confirmed via the below command. 
You need to explicitly specify 'StrictAffinity' if you wish to assign your own IP addresses in AKS>
```
./calicoctl ipam show --show-configuration
```

Since we are using Azure-CNI (and not Calico CNI), we cannot see the IPAM (IP Address Management) data.
Calico's CNI would be able to expose the IP usage as well as IP allocation within CIDR's and IP Groups.
```
./calicoctl ipam show --show-blocks
```

The same can be said for encapsulation vs. non-ecap mode chosen for those IP addresses.
Since there is no Calico CNI, we won't be able to confirm whether VXLAN or IPIP mode are enabled.
```
./calicoctl get ippool -o wide
```

Confirm all Azure CNI pods are running within the kube-system namespace:
```
kubectl get pods -n kube-system
```
