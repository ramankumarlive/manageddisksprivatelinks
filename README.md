# manageddisksprivatelinks

subscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
resourceGroupName=privatelinkstesting 
region=CentralUSEUAP
diskAccessName=myDiskAccessForPrivateLinks
diskNameForUpload=mdForUploadWithPrivateLinks
diskSizeInBytes=136367309312
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

az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateEmptyDiskForUpload.json" \
--parameters "diskName=$diskNameForUpload" "diskAccessId=$diskAccessId" "region=$region" "networkAccessPolicy=AllowPrivate" "diskSkuName=Standard_LRS" "uploadSizeBytes=$diskSizeInBytes"

region=CentralUSEUAP
resourceGroupName=privatelinkstesting 
vnetName=myVNETForPrivateLinks
subnetName=mysubnetForPrivateLinks
vmName=vminsubnetwpl
publicIpAddressName=${vmName}ip
vmSize=Standard_DS3_V2
image=win2019datacenter 
adminUserName=ramankum
adminPassword=@Password123

az vm create -g $resourceGroupName -n $vmName -l $region --image $image --size $vmSize \
--vnet-name $vnetName --subnet $subnetName --public-ip-address-dns-name $publicIpAddressName \
--admin-username $adminUserName --admin-password $adminPassword


osDiskId=$(az vm show -g $resourceGroupName -n $vmName -l $region --query 'storageProfile.osDisk.managedDisk.id' -o tsv)

az snapshot -g $resourceGroupName -l $region 

region=CentralUSEUAP
resourceGroupName=privatelinkstesting 
vnetName=myVNETWOPL
subnetName=mysubnetWOPL
vmName=vminsubnetwopl
publicIpAddressName=${vmName}ip
vmSize=Standard_DS3_V2
image=win2019datacenter 
adminUserName=ramankum
adminPassword=@Password123

az vm create -g $resourceGroupName -n $vmName -l $region --image $image --size $vmSize \
--vnet-name $vnetName --subnet $subnetName --public-ip-address-dns-name $publicIpAddressName \
--admin-username $adminUserName --admin-password $adminPassword