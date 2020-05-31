# Private Links for exporting/importing managed disks

This repository contains samples templates and scripts for setting up and testing private links for exporting/importing Managed Disks.

## Overview
Customers can export from or import to Managed Disks without attaching them to a VM. If a user has sufficient permissions, he/she can generate a Shared Access Signature (SAS) URI of the underlying VHD of Managed Disks and then use the SAS URI to download/upload VHD. A SAS URI consists of the FQDN of the underlying hidden storage account where VHD is stored as a page blob and query parameters comprising the SAS token that has all of the information necessary to grant read/write access to the VHD. SAS gives persistent and unaudited access to the disks without any trace. Customers are worried that a malicious employee can download their data from home without any trace.  

Networking Resource Provider (NRP) provides a mechanism, named [Private Links](https://docs.microsoft.com/en-us/azure/private-link/private-link-overview) that allows an onboarded Azure services such as Storage accounts, Azure SQL to associate a resource to a NRP private endpoint resource (associated to a VNET/subnet). Once the private endpoint is associated, network reads from the private endpoint are tagged with the ID of the private link. This lets the service filter out reads/writes from outside the customer's private endpoint.  Moreover, Traffic between customers' virtual network and the service travels the Microsoft backbone network. Customers are not required to expose their services to the public internet. 

Now you can leverage Private Links for restricting the export of VHD from or the import of VHD to Managed Disks from your Azure VNET only. You can create an instance of the new resource called DiskAccess, and link it to your VNET in the same subscription by creating a private endpoint. A DiskAccess object owns a special hidden Storage account (prefixed md-impexp). The FQDN of the Storage account is mapped to a private DNS in your VNET. Any calls to the Storage account from the VNET is routed via the private DNS over the Microsoft backbone network to the Storage account. 

Disk users must associate a disk with an instance of DiskAccess to export/import the data via Private Links. Disk RP uses an already existing md-impexp Storage account associated with a private link or provisions new md-impexp storage accounts under the same private link if many disks are exported at the same time. When Disk RP provisions a new md-impexp storage account, it will inform NRP about it which will map the FQDN of the storage account with a private DNS in customers’ VNET.

## Setting up Private Links for exporting/importing Managed Disks using Azure CLI

```cli
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

# Create a DiskAccess object using an ARM template
az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
--parameters "region=$region" "diskAccessName=$diskAccessName"

diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

az network vnet create --name $vnetName --resource-group $resourceGroupName --subnet-name $subnetName

#Network policies like network security groups (NSG) are not supported for private endpoints. In order to deploy a Private Endpoint on a #given subnet, an explicit disable setting is required on that subnet. 
az network vnet subnet update --name $subnetName --resource-group $resourceGroupName --vnet-name $vnetName --disable-private-endpoint-network-policies true

#Create a private endpoint for the DiskAccess object
az network private-endpoint create --name $privateEndPointName --resource-group $resourceGroupName --vnet-name $vnetName  --subnet $subnetName --private-connection-resource-id $diskAccessId --group-ids disks --connection-name $privateEndPointConnectionName

#Create a private dns zone to store the mapping of the hidden Storage account's FQDN and the private DNS
az network private-dns zone create --resource-group $resourceGroupName --name "privatelink.blob.core.windows.net"

#To resolve the records of a private DNS zone from your virtual network, you must link the virtual network with the zone. Linked virtual #networks have full access and can resolve all DNS records published in the private zone. 
az network private-dns link vnet create --resource-group $resourceGroupName --zone-name  "privatelink.blob.core.windows.net" --name yourDNSLink --virtual-network $vnetName --registration-enabled false 

#Network Interface associated with the private endpoint has the information about the private DNS and the FQDN of the Storage account
networkInterfaceId=$(az network private-endpoint show --name $privateEndPointName --resource-group $resourceGroupName --query 'networkInterfaces[0].id' -o tsv)

#Get the FQDN of the Storage account
fqdnOfStorageAccountUsedForExportImport=$(az network nic show --ids $networkInterfaceId --query 'ipConfigurations[0].privateLinkConnectionProperties.fqdns[0]' -o tsv)

#Get the Storage account name from the FQDN
storageAccountUsedForExportImport=`echo $fqdnOfStorageAccountUsedForExportImport | cut -d'.' -f1`
privateIPAddress=$(az network nic show --ids $networkInterfaceId --query 'ipConfigurations[0].privateIpAddress' -o tsv)

#Create the DNS record 
az network private-dns record-set a create --name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName  

az network private-dns record-set a add-record --record-set-name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName -a $privateIPAddress
```

## Prerequisites for testing Private Links for exporting/importing Managed Disks using Azure CLI

1. Creat a VM in the same VNET and Subnet that has the private endpoint to the DiskAccess object.

   a. Create the VM using CLI
   ```cli
   region=CentralUSEUAP
   resourceGroupName=privatelinkstesting 

   vnetNameWithPL=myVNETForPrivateLinks
   subnetNameWithPL=mysubnetForPrivateLinks
   vmNameInSubnetWithPL=vminsubnetwpl
   publicIPNameForVMInSubnetWithPL=${vmNameInSubnetWithPL}ip
   vmSize=Standard_DS3_V2

   imagePublisher=MicrosoftWindowsServer
   imageOffer=WindowsServer
   # Use a smaller image to make the testing faster 
   imageSku=2019-Datacenter-smalldisk 
   adminUserName=ramankum
   adminPassword=@Password123
   diskAccessName=myDiskAccessForPrivateLinks

   latestImageVersion=$(az vm image list -p $imagePublisher -s $imageSku --all \
   --query "[?offer=='$imageOffer'].version" -o tsv | sort -u | tail -n 1)

   imageURN=${imagePublisher}:${imageOffer}:${imageSku}:${latestImageVersion}

   az vm create -g $resourceGroupName -n $vmNameInSubnetWithPL -l $region --image $imageURN --size $vmSize --data-disk-sizes-gb 128 \
   --vnet-name $vnetNameWithPL --subnet $subnetNameWithPL --public-ip-address-dns-name $publicIPNameForVMInSubnetWithPL \
   --admin-username $adminUserName --admin-password $adminPassword

   computerName=${publicIPNameForVMInSubnetWithPL}.${region}.cloudapp.azure.com

   echo $computerName
   ```
   b. Login into the VM by following the instructions below. Please replace the *ComputerName* with the text returned in the above step.

      1. Press Windows+R
      2. Copy mstsc /v:*computerName* in the Open text box 
      3. Hit enter

   c. Expand the C drive (OS volume) by following instructions below. You will need space to copy VHD of Managed Disks.

      1. Select and hold (or right-click) the Start button and then select Windows PowerShell (Admin).
      2. Enter the following command to resize the volume to the maximum size, specifying the drive letter of the volume you want to extend in the $drive_letter variable:

      ```PowerShell
      $size = (Get-PartitionSupportedSize -DriveLetter "C")
      Resize-Partition -DriveLetter "C" -Size $size.SizeMax
      ```

   d. Download AzCopy tool by following the instructions below #####
      1. Press Windows+R
      2. Copy https://aka.ms/downloadazcopy-v10-windows in the Open text box 
      3. Hit enter

2. Creat a VM in a new VNET and Subnet that has no private endpoint associated with the DiskAccess object

   a. Create the VM using CLI
   ```cli
   region=CentralUSEUAP
   resourceGroupName=privatelinkstesting 

   vnetNameWithoutPL=myVNETWithoutPrivateLinks
   subnetNameWithoutPL=mySubnetWithoutPrivateLinks
   vmNameInSubnetWithoutPL=vminsubnetwopl
   publicIPNameForVMInSubnetWithoutPL=${vmNameInSubnetWithoutPL}ip

   vmSize=Standard_DS3_V2
   imagePublisher=MicrosoftWindowsServer
   imageOffer=WindowsServer
   imageSku=2019-Datacenter-smalldisk 
   adminUserName=ramankum
   adminPassword=@Password123


   az vm create -g $resourceGroupName -n $vmNameInSubnetWithoutPL -l $region --image $imageURN --size $vmSize --os-disk-size-gb 128 \
   --vnet-name $subnetNameWithoutPL --subnet $subnetNameWithoutPL --public-ip-address-dns-name $publicIPNameForVMInSubnetWithoutPL \
   --admin-username $adminUserName --admin-password $adminPassword

   computerName=${publicIPNameForVMInSubnetWithoutPL}.${region}.cloudapp.azure.com

   echo $computerName
   ```
   b. Login into the VM by following the instructions below. Please replace the *ComputerName* with the text returned in the above step.

      1. Press Windows+R
      2. Copy mstsc /v:*computerName* in the Open text box 
      3. Hit enter

   c. Expand the C drive (OS volume) by following instructions below. You will need space to copy VHD of Managed Disks.

      1. Select and hold (or right-click) the Start button and then select Windows PowerShell (Admin).
      2. Enter the following command to resize the volume to the maximum size, specifying the drive letter of the volume you want to extend in the $drive_letter variable:

      ```PowerShell
      $size = (Get-PartitionSupportedSize -DriveLetter "C")
      Resize-Partition -DriveLetter "C" -Size $size.SizeMax
      ```

   d. Download AzCopy tool by following the instructions below. You will use the tool to download VHD from or upload VHD to managed disks
      1. Press Windows+R
      2. Copy https://aka.ms/downloadazcopy-v10-windows in the Open text box 
      3. Hit enter

3. Create a snapshot of the OS disk of the VM created in step #1 by associating the snapshot with the DiskAccess Object and setting the networkAccessPolicy to AllowPrivate. You will export (download) the VHD of the snapshot to the VM created in prerequisite #1 in the same Subnet as the private endpoint associated with the DiskAccess object. 
   ```cli
   diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

   osDiskId=$(az vm show -g $resourceGroupName -n $vmNameInSubnetWithPL --query 'storageProfile.osDisk.managedDisk.id' -o tsv)
   osDiskName=$(az vm show -g $resourceGroupName -n $vmNameInSubnetWithPL --query 'storageProfile.osDisk.name' -o tsv)
   snapshotNameSecuredWithPL=${osDiskName}_snapshot1

   az group deployment create -g $resourceGroupName \
      --template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateSnapshotWithExportViaPrivateLink.json" \
      --parameters "snapshotNameSecuredWithPL=$snapshotNameSecuredWithPL" "sourceResourceId=$osDiskId" "diskAccessId=$diskAccessId" "region=$region" "networkAccessPolicy=AllowPrivate" 

   ```
4. Create another snapshot of the OS disk without associating it with the DiskAccess object. You will download the VHD of the snapshot in the VM created in prerequisite #2 in the Subnet without the private endpoint. You will try to upload the VHD to a disk associated with the DiskAccess object, which will result in failure.
   ```cli
   snapshotNameNotSecuredWithPL=${osDiskName}_snapshot2
   az snapshot create -n $snapshotNameNotSecuredWithPL -g $resourceGroupName --source $osDiskId
   ```

## Test that the VHD of a snapshot associated with a DiskAccess object can be exported (downloaded) only from a VM in the Subnet with the private endpoint associated with the DiskAccess object
1. Generate the SAS URI for the snapshot created in prerequisite #3
   ```cli
   az snapshot grant-access -n $snapshotNameSecuredWithPL -g $resourceGroupName --duration-in-seconds 86400 --query [accessSas] -o tsv
   ```
2. Replace the *<SAS URI>* text below with the SAS URI in the previous step
 
   AzCopy.exe copy "*<SAS URI>*" "C:\vhd\vhdfromsnapshot.vhd" --check-md5=LogOnly
 
3. Run the AzCopy command in the VM created in prerequisite step #1. You should be able to export (download) the VHD.
4. Run the AzCopy command in the VM created in prerequisite step #2. You should *NOT* be able to export (download) the VHD.

## Test that a VHD can be uploaded to a disk associated with a DiskAccess object only from a VM in a Subnet with the private endpoint associated with the DiskAccess object
1. Generate the SAS URI for the snapshot created in prerequisite #4 and copy it. 
   ```cli
   az snapshot grant-access -n $snapshotNameNotSecuredWithPL -g $resourceGroupName --duration-in-seconds 86400 --query [accessSas] -o tsv
   ```
2. Replace the *<SAS URI>* text below with the SAS URI in the previous step
 
   AzCopy.exe copy "*<SAS URI>*" "C:\vhd\vhdfromsnapshot.vhd" --check-md5=LogOnly
 
3. Run the AzCopy command from the previous step inside the VM created in prerequisite step #2. You should be able to export (download) the VHD.

4. Create a disk for upload by associating it with the DiskAccess Object and setting the networkAccessPolicy to AllowPrivate.
   ```cli
   diskNameForUpload=myDiskForUploadWithPL

   diskSizeInBytes=$(az snapshot show -n $snapshotNameNotSecuredWithPL -g $resourceGroupName --query [diskSizeBytes] -o tsv)

   az group deployment create -g $resourceGroupName \
   --template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateEmptyDiskForUploadViaPrivateLink.json" \
   --parameters "diskName=$diskNameForUpload" "diskAccessId=$diskAccessId" "region=$region" "networkAccessPolicy=AllowPrivate" "diskSkuName=Standard_LRS" "uploadSizeBytes=$(($diskSizeInBytes+512))"
   ```
5. Generate a writable SAS URI of the disk and copy it. 
   ```cli
      az disk grant-access -n $diskNameForUpload -g $resourceGroupName --duration-in-seconds 86400 --access-level Write --query [accessSas] -o tsv
    ```
6. Replace the *<SAS URI>* text below with the SAS URI generated in the previous step
 
   AzCopy.exe copy "C:\vhd\vhdfromsnapshot.vhd" "*<SAS URI>*" 
 
7. Run the AzCopy command from the previous step inside the VM created in prerequisite step #1. You should be able to import (upload) the VHD to the disk.
8. Run the AzCopy command again inside the VM created in prerequisite step #2. You should *NOT* be able to import (upload) the VHD to the disk.


## Test the Manual Approval of Private Endpoint Connection.

 ```cli
 subscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
 resourceGroupName=privatelinkstesting 
 region=CentralUSEUAP
 diskAccessName=myDiskAccessForPLWithManualApproval
 vnetName=myVNETForPLWithManualApproval
 subnetName=mysubnetForPLWithManualApproval
 privateEndPointName=myPLWithManualApproval
 privateEndPointConnectionName=myPLConnectionWithManualApproval

 az account set --subscription $subscriptionId

 az group deployment create -g $resourceGroupName \
 --template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
 --parameters "region=$region" "diskAccessName=$diskAccessName"

 diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

 az network vnet create --name $vnetName --resource-group $resourceGroupName --subnet-name $subnetName

 az network vnet subnet update --name $subnetName --resource-group $resourceGroupName --vnet-name $vnetName --disable-private-endpoint-network-policies true

 az network private-endpoint create --name $privateEndPointName --resource-group $resourceGroupName --vnet-name $vnetName  --subnet $subnetName \
 --private-connection-resource-id $diskAccessId --group-ids disks \
 --connection-name $privateEndPointConnectionName --manual-request true --request-message 'Please approve my request'
  ```
 
## Test Private Endpoint Connection in a different subscription.
  ```cli
  diskAccessSubscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
  vnetSubscriptionId=6492b1f7-f219-446b-b509-314e17e1efb0
  resourceGroupName=privatelinkstestingCrosSub 
  region=CentralUSEUAP
  diskAccessName=myDiskAccessForPrivateLinks
  vnetName=myVNETForPrivateLinks
  subnetName=mysubnetForPrivateLinks
  privateEndPointName=myPrivateLinkForSecureMDExportImport
  privateEndPointConnectionName=myPrivateLinkConnection

  az account set --subscription $diskAccessSubscriptionId

  az group create -n $resourceGroupName -l $region

  az group deployment create -g $resourceGroupName \
  --template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
  --parameters "region=$region" "diskAccessName=$diskAccessName"

  diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

  az account set --subscription $vnetSubscriptionId

  az group create -n $resourceGroupName -l $region

  az network vnet create --name $vnetName --resource-group $resourceGroupName --subnet-name $subnetName

  az network vnet subnet update --name $subnetName --resource-group $resourceGroupName --vnet-name $vnetName --disable-private-endpoint-network-policies true

  #This step should throw an error that Managed Disks doesn't support private endpoint connections across subscriptions.
  az network private-endpoint create --name $privateEndPointName --resource-group $resourceGroupName --vnet-name $vnetName  --subnet $subnetName --private-connection-resource-id $diskAccessId --group-ids disks --connection-name $privateEndPointConnectionName
  ```
