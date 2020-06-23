# Private links for securely exporting/importing data to Azure Managed Disks

You can generate a timebound Shared Access Signature (SAS) URI for unattached managed disks and snapshots for exporting the data to other region for regional expansion, disaster recovery and to read the data for forensic analysis. You can also use the SAS URI to directly upload VHD to an empty disk from your on-premises.  Now you can leverage Private Links for restricting the export and import to Managed Disks only from your Azure VNET. Moreover, you are rest assured that the data never goes over the public internet and always travels within the secure Microsoft backbone network when you use Private Links. 

You can create an instance of the new resource called DiskAccess and link it to your VNET in the same subscription by creating a private endpoint. You must associate a disk or a snapshot with an instance of DiskAccess for exporting and importing the data via Private Links. Also, you must set the NetworkAccessPolicy property of the disk or the snapshot to AllowPrivate. 

You can set the NetworkAccessPolicy property to DenyAll to prevent anybody from generating the SAS URI for a disk or a snapshot. The default value for the NetworkAccessPolicy property is AllowAll. 


