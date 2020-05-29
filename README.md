# manageddisksprivatelinks

subscriptionId=dd80b94e-0463-4a65-8d04-c94f403879dc
resourceGroupName=privatelinkstesting 
region=CentralUSEUAP
diskAccessName=myDiskAccessForPrivateLinks

az account set --subscription $subscriptionId
az group deployment create -g $resourceGroupName \
--template-uri "https://raw.githubusercontent.com/ramankumarlive/manageddisksprivatelinks/master/CreateDiskAccess.json" \
--parameters "region=$region" "diskAccessName=$diskAccessName"
