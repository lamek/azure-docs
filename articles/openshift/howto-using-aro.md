---
title: using `az aro` | Microsoft Docs
description: Create a private cluster with Azure Red Hat OpenShift 3.11
author: klamenzo 
ms.author: b-lejaku
ms.service: container-service
ms.topic: conceptual
ms.date: 03/02/2020
keywords: aro, openshift, privatecluster, red hat
#Customer intent: As a customer, I want to create a private cluster on ARO OpenShift.
---

# Using `az aro`

The `az aro` extension allows you to create, access, and delete Azure Red Hat OpenShift clusters directly from the command line using the Azure CLI.

The `az aro` extension can be used with a whitelisted subscription against the pre-GA Azure Red Hat OpenShift v4 service, or it can be used against a development RP running at https://localhost:8443/ by setting `RP_MODE=development`.

## Installing the extension

1. Install a supported version of [Python](https://www.python.org/downloads), if
   you do not have one installed already.  The `az` client supports Python 2.7
   and Python 3.5+.  A recent Python 3.x version is recommended.  You will also
   need the `setuptools` Python package  installed.

1. Install the
   [`az`](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) client,
   if you have not already.  You will need `az` version 2.0.72 or greater, as
   this version includes the `az network vnet subnet update
   --disable-private-link-service-network-policies` flag.

1. Log in to Azure:

   ```bash
   az login
   ```

1. Git clone this repository to your local machine:

   ```bash
   git clone https://github.com/Azure/ARO-RP
   cd ARO-RP
   ```

   Note: you will be able to update the `az aro` extension in the future by
   simply running `git pull`.

1. Build the development `az aro` extension:

   `make az`

   Note: you may see a message like the following; if so you can safely ignore
   it:

   ```
   byte-compiling build/bdist.linux-x86_64/egg/azext_aro/vendored_sdks/azure/mgmt/redhatopenshift/v2019_12_31_preview/models/_models_py3.py to _models_py3.pyc
     File "build/bdist.linux-x86_64/egg/azext_aro/vendored_sdks/azure/mgmt/redhatopenshift/v2019_12_31_preview/models/_models_py3.py", line 45
       def __init__(self, *, visibility=None, url: str=None, ip: str=None, **kwargs) -> None:
                        ^
    SyntaxError: invalid syntax
    ```

1. Add the ARO extension path to your `az` configuration:

   ```bash
   cat >>~/.azure/config <<EOF
   [extension]
   dev_sources = $PWD/python
   EOF
   ```

1. Verify the ARO extension is registered:

   ```bash
   az -v
   ...
   Extensions:
   aro                                0.1.0 (dev) /path/to/rp/python/az/aro
   ...
   Development extension sources:
       /path/to/rp/python
   ...
   ```


## Registering the resource provider

If using the pre-GA Azure Red Hat OpenShift v4 service, ensure that the
`Microsoft.RedHatOpenShift` resource provider is registered:

```bash
az provider register -n Microsoft.RedHatOpenShift --wait
```


## Prerequisites to create an Azure Red Hat OpenShift v4 cluster

You will need the following in order to create an Azure Red Hat OpenShift v4
cluster:

1. A vnet containing two empty subnets, each with no network security group
   attached.  Your cluster will be deployed into these subnets.

   ```bash
   LOCATION=eastus
   RESOURCEGROUP="v4-$LOCATION"
   CLUSTER=cluster

   az group create -g "$RESOURCEGROUP" -l $LOCATION
   az network vnet create \
     -g "$RESOURCEGROUP" \
     -n dev-vnet \
     --address-prefixes 10.0.0.0/9 \
     >/dev/null
   for subnet in "$CLUSTER-master" "$CLUSTER-worker"; do
     az network vnet subnet create \
       -g "$RESOURCEGROUP" \
       --vnet-name dev-vnet \
       -n "$subnet" \
       --address-prefixes 10.$((RANDOM & 127)).$((RANDOM & 255)).0/24 \
       --service-endpoints Microsoft.ContainerRegistry \
       >/dev/null
   done
   az network vnet subnet update \
     -g "$RESOURCEGROUP" \
     --vnet-name dev-vnet \
     -n "$CLUSTER-master" \
     --disable-private-link-service-network-policies true \
     >/dev/null
   ```

1. A cluster AAD application (client ID and secret) and service principal, or
   sufficient AAD permissions for `az aro create` to create these for you
   automatically.

1. The RP service principal and cluster service principal must each have the
   Contributor role on the cluster vnet.  If you have the "User Access
   Administrator" role on the vnet, `az aro create` will set up the role
   assignments for you automatically.


## Using the extension

1. Create a cluster:

   ```bash
   az aro create \
     -g "$RESOURCEGROUP" \
     -n "$CLUSTER" \
     --vnet dev-vnet \
     --master-subnet "$CLUSTER-master" \
     --worker-subnet "$CLUSTER-worker"
   ```

   Note: cluster creation takes about 35 minutes.

1. Access the cluster console:

   You can find the cluster console URL (of the form
   `https://console-openshift-console.apps.<random>.<location>.aroapp.io/`) in
   the Azure Red Hat OpenShift v4 cluster resource:

   ```bash
   az aro list -o table
   ```

   You can log into the cluster using the `kubeadmin` user.  The password for
   the `kubeadmin` user can be found as follows:

   ```bash
   az aro list-credentials -g "$RESOURCEGROUP" -n "$CLUSTER"
   ```

1. Delete a cluster:

   ```bash
   az aro delete -g "$RESOURCEGROUP" -n "$CLUSTER"

   # (optionally)
   for subnet in "$CLUSTER-master" "$CLUSTER-worker"; do
     az network vnet subnet delete -g "$RESOURCEGROUP" --vnet-name dev-vnet -n "$subnet"
   done
   ```






# Create a private cluster with Azure Red Hat OpenShift 3.11

> [!IMPORTANT]
> ARO private clusters are currently only available in private preview in East US 2. Private preview acceptance is by invitation only. Please be sure to register your subscription before attempting to enable this feature.

Private clusters provide the following benefits:

* Private clusters do not expose cluster control plane components (such as the API servers) on a public IP address.
* The VNet of a private cluster is configurable by customers, allowing you to set up networking to allow peering with other VNets, including ExpressRoute environments. You can also configure custom DNS on the VNet in order to integrate with internal services.



## Before you begin

> [!NOTE]
> This feature requires version 2019-10-27-preview of the ARO HTTP API. It is not yet supported in the Azure CLI.

The fields in the following configuration snippet are new and must be included in your cluster configuration. `managementSubnetCidr` must be within the cluster VNet and is used by Azure to manage the cluster.

```
properties:
 networkProfile:
   managementSubnetCidr: 10.0.1.0/24
 masterPoolProfile:
   apiProperties:
     privateApiServer: true
```

A private cluster can be deployed using the sample scripts provided below. Once the cluster is deployed, execute the `cluster get` command and view the `properties.FQDN` property to determine the private IP address of the OpenShift API server.

The cluster VNet will have been created with permissions so that you can modify it. You can then setup networking to access the VNet (ExpressRoute, VPN, VNet peering) as required for your needs.

If you change the DNS nameservers on the cluster VNet, then you will need to issue an update on the cluster with the `properties.RefreshCluster` property set to `true` so that the VMs can be reimaged. This will allow them to pick up the new nameservers.

## Sample Configuration Scripts

### Environment

Fill in the environment variables below as using your own values.

> [!NOTE]
> The locatin must be set to `eastus2` as this is currently the only supported location.

```
export CLUSTER_NAME=
export LOCATION=eastus2
export TOKEN=$(az account get-access-token --query 'accessToken' -o tsv)
export SUBID=
export TENANT_ID=
export ADMIN_GROUP=
export CLIENT_ID=
export SECRET=
```

### private-cluster.json
Using the environment variables defined above, here is a sample cluster configuration with private cluster enabled.

```
{
   "location": "$LOCATION",
   "name": "$CLUSTER_NAME",
   "properties": {
       "openShiftVersion": "v3.11",
       "networkProfile": {
           "vnetCIDR": "10.0.0.0/8",
           "managementSubnetCIDR" : "10.0.1.0/24"
       },
       "authProfile": {
           "identityProviders": [
               {
                   "name": "Azure AD",
                   "provider": {
                       "kind": "AADIdentityProvider",
                       "clientId": "$CLIENT_ID",
                       "secret": "$SECRET",
                       "tenantId": "$TENANT_ID",
                       "customerAdminGroupID": "$ADMIN_GROUP"
                   }
               }
           ]
       },
       "masterPoolProfile": {
           "name": "master",
           "count": 3,
           "vmSize": "Standard_D4s_v3",
           "osType": "Linux",
           "subnetCIDR": "10.0.0.0/24",
           "apiProperties": {
            	"privateApiServer": true
            }
       },
       "agentPoolProfiles": [
           {
               "role": "compute",
               "name": "compute",
               "count": 1,
               "vmSize": "Standard_D4s_v3",
               "osType": "Linux",
               "subnetCIDR": "10.0.0.0/24"
           },
           {
               "role": "infra",
               "name": "infra",
               "count": 3,
               "vmSize": "Standard_D4s_v3",
               "osType": "Linux",
               "subnetCIDR": "10.0.0.0/24"
           }
       ],
       "routerProfiles": [
           {
               "name": "default"
           }
       ]
   }
}
```
 ## Deploy a private cluster
```
az group create --name $CLUSTER_NAME --location $LOCATION
 
cat private-cluster.json | envsubst | curl -v -X PUT \
-H 'Content-Type: application/json; charset=utf-8' \
-H 'Authorization: Bearer '$TOKEN'' -d @- \
 https://management.azure.com/subscriptions/$SUBID/resourceGroups/$CLUSTER_NAME/providers/Microsoft.ContainerService/openShiftManagedClusters/$CLUSTER_NAME?api-version=2019-10-27-preview
```

## Diagrams

### Private Cluster Architecture

![](diagrams/private-cluster-diagram.png)

### Data Flow

![](diagrams/private-cluster-data-flow.png))
