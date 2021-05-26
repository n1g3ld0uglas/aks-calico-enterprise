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

AKS cluster created using the Azure CLI are created with transparent mode by default. 
Therefore, we do not need to add a feature flag for transparent mode (ie: --transparent-mode enable)

However, we must ensure the cluster is started with the option (--network-plugin azure).
This will enable 'azure-cni' network plugin - currently, the only supported option transparent networking mode:
https://docs.microsoft.com/en-us/azure/aks/faq#what-is-azure-cni-transparent-mode-vs-bridge-mode

```
az aks create --resource-group nigelResourceGroup --name nigelAKSCluster --node-vm-size Standard_B2ms --node-count 3 --zones 1 2 3 --network-plugin azure --network-policy none
```

In Transparent Mode, Azure CNI will create and add host-side pod veth pair interfaces that will be added to the host network. 
Intra VM Pod-to-Pod communication is through ip routes that the CNI will add. 
Essentially, Pod-to-Pod communication is over layer 3 and pod traffic is routed by L3 routing rules.

<img width="667" alt="Screenshot 2021-05-26 at 10 24 43" src="https://user-images.githubusercontent.com/82048393/119646172-c34b9300-be16-11eb-951e-21b7bd8310d8.png">

The final flag we selected was to have Network policy not set ( --network-policy none )
This avoids conflicts between other network policy providers in the cluster and Calico Enterprise.
https://docs.microsoft.com/en-us/azure/aks/use-network-policies


Calico CNI is currently not supported in with transparent mode in AKS - this will break the install process is chosen.
Before proceeding, I would merge "nigelAKSCluster" as the current context within /home/nigel/.kube/config

```
az aks get-credentials --resource-group nigelResourceGroup --name nigelAKSCluster
```

I also confirm the nodes are in each of the failire zones specified within my cluster deployment:
```
kubectl describe nodes | grep -e "Name:" -e "failure-domain.beta.kubernetes.io/zone"
```

<img width="774" alt="Screenshot 2021-05-26 at 10 32 25" src="https://user-images.githubusercontent.com/82048393/119646449-17ef0e00-be17-11eb-8546-40a558e63438.png">


The goal is to have each node in it's own zone within the same region, in the case where the one zone fails:
```
kubectl get nodes -o custom-columns=NAME:'{.metadata.name}',REGION:'{.metadata.labels.topology\.kubernetes\.io/region}',ZONE:'{metadata.labels.topology\.kubernetes\.io/zone}'
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 33 54" src="https://user-images.githubusercontent.com/82048393/119646586-3ce38100-be17-11eb-97f1-24f996c3ac99.png">


If you ever need planned downtime for the cluster, you can easily scale down the node count to 0 - revert back to 3 nodes when needed:
```
az aks scale --resource-group nigelResourceGroup -name nigelAKSCluster -node-count 0
```

Now for Calico Enterprise. We will start by configuring the StorageClass.
StorageClass tells Calico Enterprise to use the GCE Persistent Disks for log storage:

```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/netpolTest/main/sc/afs.yaml
```

<img width="814" alt="Screenshot 2021-05-26 at 10 37 05" src="https://user-images.githubusercontent.com/82048393/119646709-5f759a00-be17-11eb-96df-b6d04eb44595.png">


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

<img width="1230" alt="Screenshot 2021-05-26 at 10 41 05" src="https://user-images.githubusercontent.com/82048393/119646782-74522d80-be17-11eb-9111-e43abed9f6a8.png">


Install the AKS-specific custom resources
```
kubectl create -f https://docs.tigera.io/manifests/aks/custom-resources.yaml
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 41 52" src="https://user-images.githubusercontent.com/82048393/119646830-83d17680-be17-11eb-8036-108ee735a033.png">


Monitor progress with the following command. This can take a 4-5 mins (usually)
We are waiting on for 'apiserver' and 'calico' to show as 'true' under 'AVAILABLE'
```
watch kubectl get tigerastatus
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 44 35" src="https://user-images.githubusercontent.com/82048393/119646868-8e8c0b80-be17-11eb-8b29-75e47bdcefd9.png">


If there is any significant delay in this changed, get a status on all running pods:
```
kubectl get pods -A
```

<img width="861" alt="Screenshot 2021-05-26 at 10 45 07" src="https://user-images.githubusercontent.com/82048393/119646941-9fd51800-be17-11eb-8fd5-326e0e49b151.png">


You can 'describe' - get information from a specific pod within a namespace that is not successfully running:
```
kubectl describe pod tigera-secure-kb-8848584d7-dclwb -n tigera-kibana
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 47 00" src="https://user-images.githubusercontent.com/82048393/119647053-bbd8b980-be17-11eb-8957-d481e1f6de23.png">


Create the license file (again, provided by the team at Tigera) so we can apply the update:
```
vi license.yaml
```

In order for the remaining components to work, you must install the license provided to you by Tigera
```
kubectl apply -f license.yaml
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 51 02" src="https://user-images.githubusercontent.com/82048393/119647077-c3985e00-be17-11eb-9413-9daebae38033.png">


You can now monitor progress with the following command - this might take a few minutes to complete.
As always, you can check the staus of pod creation by regularly checking - kubectl get pods -A
```
watch kubectl get tigerastatus
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 53 59" src="https://user-images.githubusercontent.com/82048393/119647148-d6ab2e00-be17-11eb-8aac-20f02da81d1b.png">


You will start seeing the new pods with status 'ContainerCreating' after the license is accepted.
Don't be afraid to run kubectl describe command if manager is taking too long to appear:

```
kubectl describe pod tigera-manager-77b7f9d674-8rl27 -n tigera-manager
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 52 51" src="https://user-images.githubusercontent.com/82048393/119647196-e7f43a80-be17-11eb-9890-1062147dd8b0.png">


Secure Calico component communications, install this set of network policies
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies-managed.yaml
```

<img width="1230" alt="Screenshot 2021-05-26 at 10 55 05" src="https://user-images.githubusercontent.com/82048393/119647263-f6daed00-be17-11eb-89a9-7284f40b43bd.png">


To expose the manager using a load balancer, create the following service
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/netpolTest/main/sc/lb.yaml
```

<img width="810" alt="Screenshot 2021-05-26 at 10 56 36" src="https://user-images.githubusercontent.com/82048393/119647339-0eb27100-be18-11eb-8a4a-48b0eeae2cb6.png">


After creating the service, it may take a few minutes for the load balancer to be created.
You can access Calico Enterprise via 'EXTERNAL-IP:PORT(443)'
```
kubectl get services -n tigera-manager tigera-manager-external
```

<img width="803" alt="Screenshot 2021-05-26 at 11 49 42" src="https://user-images.githubusercontent.com/82048393/119647740-8d0f1300-be18-11eb-8005-acf009234d81.png">


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

<img width="1230" alt="Screenshot 2021-05-26 at 11 02 45" src="https://user-images.githubusercontent.com/82048393/119648120-f98a1200-be18-11eb-9a8f-7c3302c39ff2.png">

<img width="1230" alt="Screenshot 2021-05-26 at 11 04 02" src="https://user-images.githubusercontent.com/82048393/119648258-26d6c000-be19-11eb-91f1-6edf7831c17f.png">

<img width="1230" alt="Screenshot 2021-05-26 at 11 04 15" src="https://user-images.githubusercontent.com/82048393/119648336-3eae4400-be19-11eb-9c52-ecd1c2cf956a.png">


Log into the web UI with the above base64 encoded token.
This is the basic authentication method - however, you can integrate with your idP for more secure authentication.

<img width="1230" alt="Screenshot 2021-05-26 at 11 04 58" src="https://user-images.githubusercontent.com/82048393/119648388-4d94f680-be19-11eb-9bf5-d38f55e3996a.png">


Since you ran the below 'tigera-policies-managed.yaml' file already, you should have an 'allow-tigera' tier created.
This tier is responsible for ensuring all Calico/Tigera componets work smoothly. It has a higher precendence over all tier down to the 'default' tier.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies-managed.yaml
```

<img width="883" alt="Screenshot 2021-05-26 at 11 56 46" src="https://user-images.githubusercontent.com/82048393/119648563-8339df80-be19-11eb-9da1-953eabec6742.png">


You can access Kibana via your 'EXTERNAL-IP:9443/tigera-kibana/login'
The username is 'elastic' by default, and the password token can be generated via the below command:

```
kubectl -n tigera-elasticsearch get secret tigera-secure-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' && echo
```

<img width="1230" alt="Screenshot 2021-05-26 at 11 11 14" src="https://user-images.githubusercontent.com/82048393/119648645-9d73bd80-be19-11eb-9748-4ff8c79fdc7b.png">

<img width="1230" alt="Screenshot 2021-05-26 at 11 13 42" src="https://user-images.githubusercontent.com/82048393/119648739-b9775f00-be19-11eb-80ad-bb58cd7a33e8.png">



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

<img width="915" alt="Screenshot 2021-05-26 at 11 18 06" src="https://user-images.githubusercontent.com/82048393/119648832-cf851f80-be19-11eb-80f5-4f9ce427ebc7.png">

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

<img width="506" alt="Screenshot 2021-05-26 at 11 18 58" src="https://user-images.githubusercontent.com/82048393/119648958-f5122900-be19-11eb-958f-bbcaaf407f35.png">


Confirm all Azure CNI pods are running within the kube-system namespace:
```
kubectl get pods -n kube-system
```

<img width="567" alt="Screenshot 2021-05-26 at 11 26 52" src="https://user-images.githubusercontent.com/82048393/119648973-fb080a00-be19-11eb-9551-c718ddd6a3e6.png">


If you wish to delete your AKS cluster, you can run the following command:
(this can take a few minutes to complete)

```
az aks delete --resource-group nigelResourceGroup --name nigelAKSCluster
```

Finally, delete to associated Resource Group (if no longer needed):
```
az group delete --name nigelResourceGroup --location northeurope
```
<img width="677" alt="Screenshot 2021-05-26 at 13 21 56" src="https://user-images.githubusercontent.com/82048393/119658733-660b0e00-be25-11eb-85dd-9d63f5b61962.png">

