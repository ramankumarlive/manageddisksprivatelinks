# manageddisksprivatelinks

subscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
resourceGroupName=privatelinkstesting 
region=CentralUSEUAP
diskAccessName=myDiskAccessForPrivateLinks
diskNameForUpload=mdForUploadWithPrivateLinks


az account set --subscription $subscriptionId
az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
--parameters "region=$region" "diskAccessName=$diskAccessName"

diskAccessId=$(az resource show -n $diskAccessName -g $resourceGroupName --namespace Microsoft.Compute --resource-type diskAccesses --query [id] -o tsv)

az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateEmptyDiskForUpload.json" \
--parameters "diskName=$diskNameForUpload" "diskAccessId=$diskAccessId" "region=$region" "networkAccessPolicy=AllowPrivate" "diskSkuName=Standard_LRS" "uploadSizeBytes=136367309312‬"

