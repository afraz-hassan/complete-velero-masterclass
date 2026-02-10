# Complete Velero Masterclass
In this tutorial you will learn about velero. We will learn about taking kubernetes cluster's backup, and restoring the backup.
## What is Velero
Velero (formerly Heptio Ark) is an open-source tools that provides you the features to back up and restore your Kubernetes cluster resources and persistent volumes. You can run Velero with a cloud provider or on-premises. Velero lets you:

* Take backups of your cluster and restore in case of loss.
* Migrate cluster resources to other clusters.
* Replicate your production cluster to development and testing clusters.

Visit the [Official Documentation](https://velero.io/docs/v1.17/) to know more about Velero.

## Pre-Requisites for this Tutorial

Below are the things you need to check before practicing velero:

* Running Kubernetes Cluster (AKS)
* kuberctl CLI installed
* Azure CLI for Micsrosoft Azure

**Note:** We will be using linux related commands. If you are using powershell, check this file [Powershell Velero Guide](./powershell-velero-guide.md). Also, we are using Azure Kubernetes Service for backup with Velero.

## Instal Velero CLI

Let's start simply by installing Velero CLI on our linux machine.

Feel free to go to the [Release Page](https://github.com/vmware-tanzu/velero/releases/tag/v1.5.1) to find out updated versions.

Add the Velero Repository
```
curl -L -o /tmp/velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/v1.5.1/velero-v1.5.1-linux-amd64.tar.gz
```
Extract the Velero Folder
```
tar -C /tmp -xvf /tmp/velero.tar.gz
```
Move the Binary Files
```
mv /tmp/velero-v1.5.1-linux-amd64/velero /usr/local/bin/velero
```
Setup Permissions
```
chmod +x /usr/local/bin/velero
```
Verify the Installation
```
velero --help
```

## Create Storage in Azure or AWS for Backups

Now, we'll create a storage account in Azure. Keep in ming that Velero will take backup and it will store those backups in this storage. Normally, Azure Blob Storage is used for such backups.

**Azure Portal:**

Talking in terms of Azure Portal (GUI), go ahead and create a storage account and add a container acccording to your requirements through the azure portal. Or follow the below procedure to do it through azure CLI.

Pro tip: Consider creating a separate resource group for backups to keep them separated from other resources.

In the storage account blade, go to the **Access Keys** section inside azure portal and create a storage access key for that blob storage with the appropriate permissions. Don't create key from inside the blob container.

**Azure CLI**

We need to setup these environment variables:
```
AZURE_BACKUP_RESOURCE_GROUP=<backup-resource-group>
AZURE_STORAGE_ACCOUNT_NAME=<storage-account-name>
BLOB_CONTAINER=<blob-container-name>
AZURE_BACKUP_SUBSCRIPTION_ID=<azure-subscrition-id>
```
Login to Azure and Setup Account:
```
az login

# set subscription
az account set --subscription $AZURE_BACKUP_SUBSCRIPTION_ID
```
Let's create a storage account:
```
# resource group
az group create -n $AZURE_BACKUP_RESOURCE_GROUP --location WestUS

# storage account
az storage account create \
    --name $AZURE_STORAGE_ACCOUNT_NAME \
    --resource-group $AZURE_BACKUP_RESOURCE_GROUP \
    --sku Standard_GRS
```

Get the Access Key for Storage Account using azure cli:
```
AZURE_STORAGE_ACCOUNT_ACCESS_KEY=`az storage account keys list --account-name $AZURE_STORAGE_ACCOUNT_NAME --query "[?keyName == 'key1'].value" -o tsv`
```
Now create a blob container inside the storage account:
```
az storage container create -n $BLOB_CONTAINER \
  --public-access off \
  --account-name $AZURE_STORAGE_ACCOUNT_NAME \
  --account-key $AZURE_STORAGE_ACCOUNT_ACCESS_KEY

```

## Velero Deployment on Azure


We will create an Azure credential file, which will contain that access key and other related credentials by running below command:
```
cat << EOF  > /tmp/credentials-velero
AZURE_STORAGE_ACCOUNT_ACCESS_KEY=${AZURE_STORAGE_ACCOUNT_ACCESS_KEY}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF
```

Now, deploy to install the velero for azure by running below command:
```
velero install \
    --provider azure \
    --plugins velero/velero-plugin-for-microsoft-azure:v1.1.0 \
    --bucket $BLOB_CONTAINER \
    --secret-file /tmp/credentials-velero \
    --backup-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_NAME,storageAccountKeyEnvVar=AZURE_STORAGE_ACCOUNT_ACCESS_KEY,subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID \
    --use-volume-snapshots=false
```
After deployment is successful, run below command to verify:
```
kubectl get pods -n <velero-namespace>
 
kubectl logs deployment/<velero-deployment-name> -n <velero-namespace>

```

## Create a Backup Using Velero

After the velero is up and running, it's time to take the backup. In the below example we are taking the backup of **default** kubernetes namespace for testing.

```
velero backup create default-namespace-backup --include-namespaces default
```
Descrbe the backup to know what is going:
```
velero backup describe default-namespace-backup
```
Check logs;
```
velero backup logs default-namespace-backup
```

Congratulations! its working.


## Scheduling the Backup
Previously, we created the backup manually. But now we will create the backup automatically by scheduling the backups to happen at midnight (cluster time):
```
velero schedule create default-namespace-daily \
  --schedule="0 0 * * *" \
  --include-namespaces default
```
Verify the schedule is created:
```
velero schedule get

# Check the next run time:
velero schedule describe default-namespace-daily
```
The time format in above command is the **Cron Time Format**. If you don't understand the format, visit [Cron Guru](https://crontab.guru/) to convert normal time to cron time format.

## Restore the Backup

As we have taken the backup of default namespace, let's delete and restore from backup:

```
# delete all resources

kubectl delete -f kubernetes/configmaps/configmap.yaml
kubectl delete -f kubernetes/secrets/secret.yaml
kubectl delete -f kubernetes/deployments/deployment.yaml
kubectl delete -f kubernetes/services/service.yaml
```
Restore command for velero is:
```
velero restore create default-namespace-backup --from-backup default-namespace-backup
```
Describe the process and check logs:
```
# describe
velero restore describe default-namespace-backup

# logs 
velero restore logs default-namespace-backup

# verify restored items
kubectl get all
```


## Conclusion:
Hope this tutorial will help you to get started with Velero. Thanks!
