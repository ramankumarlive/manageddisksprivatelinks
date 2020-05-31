# manageddisksprivatelinks

subscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
resourceGroupName=privatelinkstesting 
region=CentralUSEUAP
diskAccessName=myDiskAccessForPrivateLinks
vnetName=myVNETForPrivateLinks
subnetName=mysubnetForPrivateLinks
privateEndPointName=myPrivateLinkForSecureMDExportImport
privateEndPointConnectionName=myPrivateLinkConnection

az account set --subscription $subscriptionId

az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
--parameters "region=$region" "diskAccessName=$diskAccessName"

diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

az network vnet create --name $vnetName --resource-group $resourceGroupName --subnet-name $subnetName
az network vnet subnet update --name $subnetName --resource-group $resourceGroupName --vnet-name $vnetName --disable-private-endpoint-network-policies true
az network private-endpoint create --name $privateEndPointName --resource-group $resourceGroupName --vnet-name $vnetName  --subnet $subnetName --private-connection-resource-id $diskAccessId --group-ids disks --connection-name $privateEndPointConnectionName
az network private-dns zone create --resource-group $resourceGroupName --name "privatelink.blob.core.windows.net"
az network private-dns link vnet create --resource-group $resourceGroupName --zone-name  "privatelink.blob.core.windows.net" --name yourDNSLink --virtual-network $vnetName --registration-enabled false 

networkInterfaceId=$(az network private-endpoint show --name $privateEndPointName --resource-group $resourceGroupName --query 'networkInterfaces[0].id' -o tsv)
fqdnOfStorageAccountUsedForExportImport=$(az network nic show --ids $networkInterfaceId --query 'ipConfigurations[0].privateLinkConnectionProperties.fqdns[0]' -o tsv)
storageAccountUsedForExportImport=`echo $fqdnOfStorageAccountUsedForExportImport | cut -d'.' -f1`
privateIPAddress=$(az network nic show --ids $networkInterfaceId --query 'ipConfigurations[0].privateIpAddress' -o tsv)

#Create DNS records 
az network private-dns record-set a create --name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName  
az network private-dns record-set a add-record --record-set-name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName -a $privateIPAddress

#Create VM for Managed Disks Export/Importing testing
region=CentralUSEUAP
resourceGroupName=privatelinkstesting 

vnetNameWithPL=myVNETForPrivateLinks
subnetNameWithPL=mysubnetForPrivateLinks
vmNameInSubnetWithPL=vminsubnetwpl
publicIPNameForVMInSubnetWithPL=${vmNameInSubnetWithPL}ip

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
diskAccessName=myDiskAccessForPrivateLinks

# Use a smaller image to make the testing faster 
latestImageVersion=$(az vm image list -p $imagePublisher -s $imageSku --all \
--query "[?offer=='$imageOffer'].version" -o tsv | sort -u | tail -n 1)

imageURN=${imagePublisher}:${imageOffer}:${imageSku}:${latestImageVersion}

diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

# Creat a VM in the same VNET and Subnet that has the private endpoint to the DiskAccess object so that secured (with private end point) disks can be exported to or imported from the VM
az vm create -g $resourceGroupName -n $vmNameInSubnetWithPL -l $region --image $imageURN --size $vmSize --data-disk-sizes-gb 128 \
--vnet-name $vnetNameWithPL --subnet $subnetNameWithPL --public-ip-address-dns-name $publicIPNameForVMInSubnetWithPL \
--admin-username $adminUserName --admin-password $adminPassword

computerName=${publicIPNameForVMInSubnetWithPL}.${region}.cloudapp.azure.com

echo $computerName

# Run the command below in a seperate command prompt by replacing the ComputerName with the text returned in the above step
mstsc /v:computerName

https://docs.microsoft.com/en-us/windows-server/storage/disk-management/extend-a-basic-volume#to-extend-a-volume-by-using-disk-management

https://aka.ms/downloadazcopy-v10-windows

# Creat a VM in a new VNET and Subnet that has no private endpoint to the DiskAccess object so that secured disks cannot be exported to or imported from the VM
az vm create -g $resourceGroupName -n $vmNameInSubnetWithoutPL -l $region --image $imageURN --size $vmSize --os-disk-size-gb 128 \
--vnet-name $subnetNameWithoutPL --subnet $subnetNameWithoutPL --public-ip-address-dns-name $publicIPNameForVMInSubnetWithoutPL \
--admin-username $adminUserName --admin-password $adminPassword

computerName=${publicIPNameForVMInSubnetWithoutPL}.${region}.cloudapp.azure.com

echo $computerName

# Run the command below in a seperate command prompt by replacing the ComputerName with the text returned in the above step
mstsc /v:computerName

https://docs.microsoft.com/en-us/windows-server/storage/disk-management/extend-a-basic-volume#to-extend-a-volume-by-using-disk-management
https://aka.ms/downloadazcopy-v10-windows

osDiskId=$(az vm show -g $resourceGroupName -n $vmNameInSubnetWithPL --query 'storageProfile.osDisk.managedDisk.id' -o tsv)
osDiskName=$(az vm show -g $resourceGroupName -n $vmNameInSubnetWithPL --query 'storageProfile.osDisk.name' -o tsv)
snapshotNameSecuredWithPL=${osDiskName}_snapshot1


az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateSnapshotWithExportViaPrivateLink.json" \
--parameters "snapshotNameSecuredWithPL=$snapshotNameSecuredWithPL" "sourceResourceId=$osDiskId" "diskAccessId=$diskAccessId" "region=$region" "networkAccessPolicy=AllowPrivate" 

az snapshot grant-access -n $snapshotNameSecuredWithPL -g $resourceGroupName --duration-in-seconds 86400 --query [accessSas] -o tsv


https://docs.microsoft.com/en-us/windows-server/storage/disk-management/extend-a-basic-volume#to-extend-a-volume-by-using-disk-management
https://aka.ms/downloadazcopy-v10-windows

AzCopy.exe copy "https://md-impexp-41tp5m52ltq1.z13.blob.storage.azure.net/320w3v2h45bt/relaycor?sv=2017-04-17&sr=b&si=9b49c2d2-7bf3-43d0-8c80-4e41e30d5da6&sig=l5UWOIvmVzbipeBZ%2B0jQfEtqaPEcEkb4%2BxasAce%2FZC8%3D" "F:\vhd\vhdfromsnapshot1.vhd" 

#Upload Testing

snapshotNameNotSecuredWithPL=${osDiskName}_snapshot2

az snapshot create -n $snapshotNameNotSecuredWithPL -g $resourceGroupName --source $osDiskId

az snapshot grant-access -n $snapshotNameNotSecuredWithPL -g $resourceGroupName --duration-in-seconds 86400 --query [accessSas] -o tsv

AzCopy.exe copy "https://md-kqqwvgsxdz31.blob.core.windows.net/hrlv3crp0tzd/abcd?sv=2017-04-17&sr=b&si=36bd314c-a182-404a-a0a8-1ac597967cb8&sig=7iygEQayJ4eINSskk4Uhxu%2Bo6WsVUw2tevBGhkclURE%3D" "C:\vhd\Win2019.vhd" 

diskNameForUpload=myDiskForUploadWithPL

diskSizeInBytes=$(az snapshot show -n $snapshotNameNotSecuredWithPL -g $resourceGroupName --query [diskSizeBytes] -o tsv)

az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateEmptyDiskForUploadViaPrivateLink.json" \
--parameters "diskName=$diskNameForUpload" "diskAccessId=$diskAccessId" "region=$region" "networkAccessPolicy=AllowPrivate" "diskSkuName=Standard_LRS" "uploadSizeBytes=$(($diskSizeInBytes+512))"

az disk grant-access -n $diskNameForUpload -g $resourceGroupName --duration-in-seconds 86400 --access-level Write --query [accessSas] -o tsv

AzCopy.exe copy "C:\vhd\Win2019.vhd" "https://md-impexp-41tp5m52ltq1.z13.blob.storage.azure.net/wx01d1hgmwb1/abcd?sv=2017-04-17&sr=b&si=7a2849eb-6742-4c83-8459-44863464a809&sig=Pzgp7DVXHWbZbi3BYEAykmTeYFpJkgomO%2B9S3CMR4RE%3D"



#Manual Approval Testing 
#Conclusion: Manual Approval is not supported today 

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

az network private-endpoint-connection approve -g $resourceGroupName -n $privateEndPointName \
--resource-name $diskAccessName --type Microsoft.Compute/diskAccesses --description "Approved"

az network private-dns zone create --resource-group $resourceGroupName --name "privatelink.blob.core.windows.net"

az network private-dns link vnet create --resource-group $resourceGroupName --zone-name  "privatelink.blob.core.windows.net" --name yourDNSLink \
--virtual-network $vnetName --registration-enabled false 

networkInterfaceId=$(az network private-endpoint show --name $privateEndPointName --resource-group $resourceGroupName --query 'networkInterfaces[0].id' -o tsv)
fqdnOfStorageAccountUsedForExportImport=$(az network nic show --ids $networkInterfaceId --query 'ipConfigurations[0].privateLinkConnectionProperties.fqdns[0]' -o tsv)
storageAccountUsedForExportImport=`echo $fqdnOfStorageAccountUsedForExportImport | cut -d'.' -f1`
privateIPAddress=$(az network nic show --ids $networkInterfaceId --query 'ipConfigurations[0].privateIpAddress' -o tsv)

#Create DNS records 
az network private-dns record-set a create --name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName  
az network private-dns record-set a add-record --record-set-name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName -a $privateIPAddress

#Create DNS records 
az network private-dns record-set a create --name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName  
az network private-dns record-set a add-record --record-set-name $storageAccountUsedForExportImport --zone-name privatelink.blob.core.windows.net --resource-group $resourceGroupName -a $privateIPAddress


#Cross Subscription Testing

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

#This step should throw an error that DiskAccess 'myDiskAccessForPrivateLinks' belongs to subscription 'dd80b94e-0463-4a65-8d04-c94f403879dc' whereas PrivateEndpoint belongs to #subscription '6492b1f7-f219-446b-b509-314e17e1efb0'. Currently Managed Disks doesn't support private endpoint connections across subscriptions.
az network private-endpoint create --name $privateEndPointName --resource-group $resourceGroupName --vnet-name $vnetName  --subnet $subnetName --private-connection-resource-id $diskAccessId --group-ids disks --connection-name $privateEndPointConnectionName

