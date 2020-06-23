# Private links for securely exporting/importing data to Azure Managed Disks

You can generate a timebound Shared Access Signature (SAS) URI for unattached managed disks and snapshots for exporting the data to other region for regional expansion, disaster recovery and to read the data for forensic analysis. You can also use the SAS URI to directly upload VHD to an empty disk from your on-premises.  Now you can leverage [Private Links](https://docs.microsoft.com/en-us/azure/private-link/private-link-overview) for restricting the export and import to Managed Disks only from your Azure VNET. Moreover, you are rest assured that the data never goes over the public internet and always travels within the secure Microsoft backbone network when you use Private Links. 

You can create an instance of the new resource called DiskAccess and link it to your VNET in the same subscription by creating a private endpoint. You must associate a disk or a snapshot with an instance of DiskAccess for exporting and importing the data via Private Links. Also, you must set the NetworkAccessPolicy property of the disk or the snapshot to AllowPrivate. 

You can set the NetworkAccessPolicy property to DenyAll to prevent anybody from generating the SAS URI for a disk or a snapshot. The default value for the NetworkAccessPolicy property is AllowAll. 

## Create an instance of DiskAccess using Azure CLI

```azurecli-interactive
subscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
resourceGroupName=privatelinkstesting
region=CentralUSEUAP
diskAccessName=myDiskAccessForPrivateLinks
vnetName=myVNETForPrivateLinks
subnetName=mysubnetForPrivateLinks
privateEndPointName=myPrivateLinkForSecureMDExportImport
privateEndPointConnectionName=myPrivateLinkConnection

az login

az account set --subscription $subscriptionId

az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
--parameters "region=$region" "diskAccessName=$diskAccessName"

diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)
```

## Create a Virtual Network

Network policies like network security groups (NSG) are not supported for private endpoints. In order to deploy a Private Endpoint on a given subnet, an explicit disable setting is required on that subnet. 

```azurecli-interactive
az network vnet create --name $vnetName --resource-group $resourceGroupName --subnet-name $subnetName
```
## Disable subnet private endpoint policies

Azure deploys resources to a subnet within a virtual network, so you need to update the subnet to disable private endpoint network policies. 

```azurecli-interactive
az network vnet subnet update --name $subnetName --resource-group $resourceGroupName --vnet-name $vnetName --disable-private-endpoint-network-policies true
```
## Create a private endpoint for the DiskAccess object

```azurecli-interactive
az network private-endpoint create --name $privateEndPointName --resource-group $resourceGroupName --vnet-name $vnetName  --subnet $subnetName --private-connection-resource-id $diskAccessId --group-ids disks --connection-name $privateEndPointConnectionName
```

## Configure the Private DNS Zone

Create a Private DNS Zone for Storage blob domain, create an association link with the Virtual Network
and create a DNS Zone Group to associate the private endpoint with the Private DNS Zone. 

```azurecli-interactive
az network private-dns zone create --resource-group $resourceGroupName \ 
    --name "privatelink.blob.core.windows.net"

az network private-dns link vnet create --resource-group $resourceGroupName \
    --zone-name "privatelink.blob.core.windows.net" \
    --name yourDNSLink \
    --virtual-network $vnetName \
    --registration-enabled false 

az network private-endpoint dns-zone-group create \
   --resource-group $resourceGroupName \
   --endpoint-name $privateEndPointName \
   --name yourZoneGroup \
   --private-dns-zone "privatelink.database.windows.net" \
   --zone-name disks
```

