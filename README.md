# BatchAI Workshop

Batch AI provides managed infrastructure to help data scientists with cluster
 management and scheduling, scaling, and monitoring of AI jobs.
 Batch AI works on top of `virtual machine scale sets` and `docker`.

Batch AI can run training jobs in docker containers or directly on the compute
nodes.

## Batch AI

* Cluster
* Jobs
* Azure File Share - stdout, stderr, may contain python scripts
* Azure Blob Storage - python scripts, data

![image](https://user-images.githubusercontent.com/7232635/38520388-aed7b5ec-3c10-11e8-84e2-39a0d1a17f81.png)

## Parallelizing Batch AI jobs

* Python train and test scripts define the `parallel strategy` used, **not Batch AI**.

For example,

* `CNTK` uses a `synchronous data parallel` training strategy
* `Tensorflow` uses a `asynchronous model parallel` training strategy

## Note

* Make sure `.sh` scripts have `LF` endings - use `dos2unix` to fix
* To enable faster communication between the nodes it´s necessary to use `Intel MPI` and have `InfiniBand` on the VM
* `NC24r` (works with `Intel MPI` and `InfiniBand`) quota is `1 core` by default in any subscription, so make quota increase requests early
* There's no reset ssh-key for nodes
* Do **not** put `CMD` in the dockerfile used by Batch AI. Since the container runs in **detached mode**, it will exit on `CMD`
* Error messages within the container are not very descriptive
* Clusters take a long time to provision and deallocate

## Resources

* [Install Azure CLI 2.0 for WSL](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest)
* [Batch AI Recipes](https://github.com/Azure/BatchAI/tree/master/recipes)
* [Azure CLI Docs](https://github.com/Azure/BatchAI/blob/master/documentation/using-azure-cli-20.md)
* [Swagger Docs for Batch AI](https://editor.swagger.io//?_ga=2.103282566.745803966.1523299917-1903704715.1523299917#/)
* [Batch AI Environment Variables](https://github.com/Azure/BatchAI/blob/master/documentation/using-batchai-environment-variables.md)
* [Setting up KeyVault](https://github.com/Azure/BatchAI/blob/master/documentation/using-azure-cli-20.md#using-keyvault-for-storing-secrets)

## Configure Azure CLI to use Batch AI

* [Azure CLI Configuration](https://github.com/Azure/BatchAI/blob/master/documentation/using-azure-cli-20.md#configuration)

## Set default subscription

```sh
az account set -s <subscription id>
az account list -o table
```

## Create resource group

```sh
az group create -n <rg name> -l eastus
```

## Create a storage account

```sh
az storage account create \
  -n <storage account name> \
  --sku Standard_LRS \
  -l eastus \
  -g <rg name>
```

## Create a file share

```sh
az storage share create \
  -n <share name> \
  --account-name <storage account name> \
  --account-key <storage account key>
```

### Get storage account key of file share

```sh
az storage account keys list \
  -n <storage account name> \
  -g <rg name> \
  --query "[0].value"
```

## Create a directory in your file share to hold python scripts

```sh
az storage directory create \
  -s <share name> \
  -n yolo \
  --account-name <storage account name> \
  --account-key <storage account key>
```

## Upload python scripts to file share

```sh
az storage file upload \
  -s <share name> \
  --source <python script> \
  -p yolo \
  --account-name <storage account name> \
  --account-key <storage account key>
```

## Create cluster

### Create a cluster.json

Config parameters defined by `ClusterCreateParameters` in the [batch ai swagger docs](https://editor.swagger.io//?_ga=2.103282566.745803966.1523299917-1903704715.1523299917#/).

* [List of VM images](https://docs.microsoft.com/en-us/azure/batch/batch-linux-nodes#list-of-virtual-machine-images)

### Create cluster with cluster.json config

```sh
az batchai cluster create \
  -n <cluster name> \
  -l eastus \
  -g <rg name> \
  -c cluster.json
```

### Create cluster without cluster.json config

```sh
az batchai cluster create \
  -n <cluster name> \
  -g <rg name> \
  -l eastus \
  --storage-account-name <storage account name> \
  --storage-account-key <storage account key> \
  -i UbuntuDSVM \
  -s Standard_NC6 \
  --min 2 \
  --max 2 \
  --afs-name <share name> \
  --afs-mount-path external \
  -u $USER \
  -k ~/.ssh/id_rsa.pub \
  -p <password>
```

### View Cluster Status

```sh
az batchai cluster show \
  -n <cluster name> \
  -g <rg name> \
  -o table
 ```

## Create a job

### Create job.json

* View `JobBaseProperties` in the [batch ai swagger docs](https://editor.swagger.io//?_ga=2.103282566.745803966.1523299917-1903704715.1523299917#/) for the possible parameters to use in `job.json`.

```sh
az batch ai job create \
  -g <rg name> \
  -l eastus \
  -n <job name> \
  -r <cluster name> \
  -c job.json
```

### Monitor the job

```sh
az batchai job show \
  -n <job name> \
  -g <rg name> \
  -o table
```

### Stream job file output

```sh
az batchai job stream-file \
  -j <job name> \
  -n stdout.txt \
  -d stdouterr \
  -g <rg name>
```

## List ip and port of nodes in cluster

```sh
az batchai cluster list-nodes \
  -n <cluster name> \
  -g <rg name>
```

## SSH into the VM

```sh
ssh <ip> -p <port>
```

`$AZ_BATCHAI_MOUNT_ROOT` is an environment variable set by Batch AI for each job, it's value depends on the image used for nodes creation. For example, on Ubuntu based images it's equal to `/mnt/batch/tasks/shared/LS_root/mounts`. You can `cd` to this directory and view the python scripts and logs.