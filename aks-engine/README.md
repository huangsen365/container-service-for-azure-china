# Microsoft aks-engine deployment guide on Azure China

[AKS Engine](https://github.com/Azure/aks-engine) provides convenient tooling to quickly bootstrap Kubernetes clusters on Azure. By leveraging [ARM (Azure Resource Manager)](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview), AKS Engine helps you create, destroy and maintain clusters provisioned with basic IaaS resources in Azure. AKS Engine is also the library used by AKS for performing these operations to provide managed service implementations.


## 1. download aks-engine binary
* take aks-engine v0.34.0 as an example 
```
aks_version=v0.34.0
wget https://mirror.azure.cn/kubernetes/aks-engine/$aks_version/aks-engine-$aks_version-linux-amd64.tar.gz
tar -xvzf aks-engine-$aks_version-linux-amd64.tar.gz
```
> as an alternative, you could also [Build aks-engine from source](https://github.com/Azure/aks-engine/blob/master/docs/acsengine.zh-CN.md)


## 2. Generate an SSH Key 
In addition to using Kubernetes APIs to interact with the clusters, cluster operators may access the master and agent machines using SSH. If you don't have an SSH key [cluster operators may generate a new one](https://github.com/Azure/aks-engine/blob/master/docs/ssh.md#ssh-key-generation).
```
ssh-keygen -t rsa
```

## 3. [Install azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
```
sudo su
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ wheezy main" | sudo tee /etc/apt/sources.list.d/azure-cli.list
apt-key adv --keyserver packages.microsoft.com --recv-keys 417A0893
apt-get install -y apt-transport-https
apt-get update
apt-get install -y azure-cli
```

## 4. Create a Service Principle
Kubernetes clusters have integrated support for various cloud providers as core functionality. On Azure, aks-engine uses a Service Principal to interact with Azure Resource Manager (ARM). Follow the instructions to [create a new service principal](https://github.com/Azure/aks-engine/blob/master/docs/serviceprincipal.md).
```
az cloud set -n AzureChinaCloud
az login
az account set --subscription="${SUBSCRIPTION_ID}" #if there is only one subscription, this step is optional
az ad sp create-for-rbac -n RBAC_NAME --role="Contributor" --scopes="/subscriptions/{subs-id}"
```

## 5. Clone & edit Kubernetes cluster definition file [example/kubernetes.json](https://raw.githubusercontent.com/Azure/aks-engine/master/examples/kubernetes.json)
Acs-engine consumes a [cluster definition](https://github.com/Azure/aks-engine/blob/master/docs/clusterdefinition.md) which outlines the desired shape, size, and configuration of Kubernetes. There are a number of features that can be enabled through the cluster definition:
* adminUsername - change username for agent nodes
* dnsPrefix - must be a region-unique name and will form part of the hostname (e.g. myprod1, staging, leapingllama) 
* keyData - must contain the public portion of an SSH key - this will be associated with the adminUsername value found in the same section of the cluster definition (e.g. 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABA....')
* clientId - this is the service principal's appId uuid or name from step 4
* secret - this is the service principal's password or randomly-generated password from step 4
* add **location** definition `"location": "chinaeast2",` behind `apiVersion: "vlabs"`
  * specify `location` as (`chinaeast`, `chinanorth`, `chinaeast2`, `chinanorth2`) in cluster defination file
* Kubernetes cluster definition example files for Azure China
  * [VM Availability Set cluster example](./example/kubernetes-1.11.5.json)
  * [VM Scale Set cluster example](./example/kubernetes-vmss-1.13.2.json)


## 6. Generate ARM templates
Run `./aks-engine generate kubernetes.json` command to generate a number of files that may be submitted to ARM. By default, generate will create a new directory(naming as `dnsPrefix`) after your cluster nested in the `_output` directory. The generated files include:
* apimodel.json - is an expanded version of the cluster definition provided to the generate command. All default or computed values will be expanded during the generate phase
* azuredeploy.json - represents a complete description of all Azure resources required to fulfill the cluster definition from apimodel.json
* azuredeploy.parameters.json - the parameters file holds a series of custom variables which are used in various locations throughout azuredeploy.json
* certificate and access config files - orchestrators like Kubernetes require certificates and additional configuration files (e.g. Kubernetes apiserver certificates and kubeconfig)

## 7. Deploy K8S cluster with ARM
[Deploy the output azuredeploy.json and azuredeploy.parameters.json](https://github.com/Azure/aks-engine/blob/master/docs/acsengine.md#deployment-usage)
```
# create a resource group first
RESOURCE_GROUP_NAME=demo-k8s
az group create -l chinaeast2 -n $RESOURCE_GROUP_NAME

# deploy ARM template
dnsPrefix=demo-k8s
az group deployment create \
    --name="$dnsPrefix" \
    --resource-group=$RESOURCE_GROUP_NAME \
    --template-file="./_output/$dnsPrefix/azuredeploy.json" \
    --parameters "@./_output/$dnsPrefix/azuredeploy.parameters.json"
```

## 8. Verify the cluster status
 - Log in to master node via SSH by 
```
ssh adminUsername@<master_node_fqdn>
```
> usually master_node_fqdn is `$dnsPrefix.$REGION.cloudapp.chinacloudapi.cn`

 - run below command
```
kubectl get services --all-namespaces
```
> If all services(like kubernetes, heapster, kube-dns, kubernetes-dashboard, tiller-deploy) in `default` and `kube-system` namespaces are working fine, it indicates the cluster were installed correctly.

## Tips
 - [Config kubernetes dashboard (only for testing purpose)](./config-k8s-dashboard.md)
 - If there is provision failure on the node, check following log file for diagnostics:
```
/var/log/azure/cluster-provision.log
```
