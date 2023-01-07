# Attaching Storage Volumes to Pods

## Introducing volumes
When a container is terminated and restarted, all changed files are lost. This is because k8s simply replaced the container instead of restart. Thus, we need to find a way to persist changes when need.this can be achieved by adding a volume to the pod and mounting it into the container.

- Mounting
  Attaching a file system of some storage device or volume into a specific location in the operating system's file tree. 
  ![Mounting a filesystem into the file tree](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image002.png)

### Demonstrating the need for volumes

- Summary
  Use the MongoDB as a test. Without the volumes, whenever the MongoDB container terminated and restarted, we will lose the data changes we did.

- :warning: Command Error
  - Connect into MongoDB container to make changes 
    - Command: `k exec -it quiz -c mongo -- mongosh`
  - Insert data into MongoDB
    - Command: `db.questions.insertOne({...})`
  - Restart the MongoDB database 
    - Command: `k exec -it -c mongo -- mongosh admin --eval "db.shutdownServer()"`
  - Cause
    - `mongo` has been replaced by `mongosh` since MongoDB 6.0. [Reference](https://stackoverflow.com/questions/73582703/mongo-command-not-found-on-mongodb-6-0-docker-container)

### Understanding how volumes fit into pods

Volumes are components within the pod and share the lifecycle with pod. 

- Overview
  ![Volumes are defined at the pod level and mounted in the pod's containers](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image005.png)

- Persisting files across container restarts 
  
  File systems in containers are ephemeral. When the container restarted, we lose the associated file system. With mounted volumes, things will look like the following: 
  ![Volumes ensure that part of the container's file system is persisted across restarts](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image006.png)

- Volumes and Containers Mapping
  - A pod can have multiple volumes.
  - A volume can be counted to multiple containers.
  - A container can mount zero or more volumes. 
  - **Volumes mounted to a container can be configured either as read/write or as read-only.**
  - **In a pod with multiple containers, some volumes can be mounted exclusively to some containers.**

- Sharing files between multiple containers
  ![A volume can be mounted into more than one container](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image008.png)

- Persisting data across pod instances 
  As we know, regular volumes share the lifecycle of the pod. In order to keep the data when pod terminated and restarted, we can use **persitent storage** outside the pod. 
  ![Pod volumes can also map to storage volumes taht persist across and restarts](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image009.png)

- Sharing data between pods
  With the external persistent storage volume, we can share the same volume between pods as the following. **NFS (Network File System) can be shared for pods deployed to different nodes.**
  ![Using volumes to share data between pods](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image010.png)

- Introducing the volume types
  - `emptyDir`: A directory created before pod starts and allows the pod to store data during its lifecycle. You can use init containers to initialize this.
  - `hostPath`: Mounting the files from the worker node's filesystem into the pod.
  - `nfs`: Network file system
  - `azureFile`, `azureDisk`, `gcePersistentDisk`, `awsElasticBlockStore`.
  - `configMap`, `secret`, `downwardAPI` and `projected` volume type: Use to configure the applicaiton in a pod to expose information about the pod or other k8s objects.
  - `persistentVolumeClain`: details in chapter 8. 
  - `csi`: Allows users to implement their own storage driver.

## Using an `emptyDir` volume

### Adding an `emptyDir` volume to a pod
  Modify the definition of the pod to add an `emptyDir` volume and mount to a container. 
  - Changes
    - Add an `emptyDir` volume in `spec` section.
    - Mount the volume to a container.
    - Overview
      ![The quiz pod with an emptyDir volume for storing MongoDB data files](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image011.png)

- Sample code 
  [pod.quiz.emptydir.yaml](deployment/pod.quiz.emptydir.yaml)

- Configuring the `emptyDir` volume
  - `medium`
    - The type of storage medium to use for the directory. Default is creating the directory on the worker node's disk. The other supported option is `Memory`, which use virtual memory filesystem.
    - Example
      ```yaml
      volumes:
        - name: content
          emptyDir:
            medium: Memory
      ```
  - `sizeLimit`
    - The total amount for the directory. For example, `10Mi` is to set the maximum size to 10 mebibytes.

### Mounting the volume in a container

- Full list of supported fields
  - `name`: The name of the volume to mount. 
  - `mountPath`: The path within the container at which to mount the volume.
  - `readOnly`: Whether to mount the volume as read-only. Defaults to `false`.
  - `mountPropagation`, `subPath` and `subPathExpr` are not so common.

- :exclamation: **Where to find the `emptyDir` volume**
  - The emptyDir is a normal file directory in the node's filesystem that's mounted into the container
    ![The emptyDir is a normal file directory in the node's filesystem that's mounted into the container](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image012.png)

  - Find the pod id
    - Command: `k get pod quiz -o json | jq .metadata.uid`

  - Log into the node using `node-shell`
    - Command: `k node-shell <node-name>`

  - Go to the path
    - Path: `/var/lib/kubelet/pods/<pod_UID>/volumes/kubernetes.io~empty-dir/<volume_name>`

### Populating an emptyDir volume with data using an init container
In this example, we are going to mount the volume to an init container first. It will then be populated by the init container. Later, the populated volume will be mounted to the main container.

- Overview of the Quiz service
  ![Using an init container to initialize an emptyDir volume](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image013.png)

- Sample pod definition
  [pod.quiz.emptydir.init.yaml](deployment/pod.quiz.emptydir.init.yaml)

- Verification
  - Command: `k exec -it quiz -c mongo -- mongosh kiada --quiet --eval "db.questions.countDocuments()"`

### Sharing files between containers
In this example, we are going to mount the same volume to multiple containers and config access (`read/write` or `readOnly`). The volume will be shared by both containers.

- Overview of the Quote service upgrade 
  ![The new version of the Quote servie uses two containers and a shared volume](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image014.png)

- Sample pod definition
  [pod.quote.yaml](deployment/pod.quote.yaml)

- Verification
  - Command
    - From container `quote-writer`: `k exec quote -c quote-writer -- cat /var/local/output/quote`
    - From container `nginx`: `k exec quote -c nginx -- cat /usr/share/nginx/html/quote`
    - From node: 
      - Get node name: `k get pod -o wide` --> You will see the node name. 
      - Connect node with `node-shell`: `k node-shell <node-name>`

## Using external storage in pods

As `emptyDir` is a folder in local node. There are many reasons that the attached volume will be messed up. 
- When the pod got deleted, the volume will be removed from node. 
- K8s could reschedule a pod to another node at any time. Then the associated volume will not be able to visit. The pod will have another volume in the new node, but we lose the previous data.

:exclamation: **With the above concerns, we need to have some sort of persisted volumes that will not affected by the lifecycle of pod and k8s reschedule.**

### Attach `AzureDisk` as an external volume

- :blue_book: [Create a static volume with Azure disks in AKS](https://learn.microsoft.com/en-us/azure/aks/azure-disk-volume)

Foolwoing are the steps and commands to create an Azure Disk. Later, we will attach the Azure Disk to a pod.
- Create Azure Disk in the nodepool resource group
  - Get resource group name
    `$nodeRG=az aks show --resource-group k8sInAction --name k8sLearn --query nodeResourceGroup -o tsv`
  - Create Azure Disk
    :exclamation: The result of the following command is the resource id of an Azure Disk. You will need it in the following steps.
    ```shell
    az disk create \
    --resource-group $nodeRG \
    --name myAKSDisk \
    --size-gb 10 \
    --query id --output tsv
    ```
  - Create `pv-azuredisk.yaml` and `pvc-azuredisk.yaml`
    - [pv-azuredisk.yaml](deployment/azureDisk/pv-azuredisk.yaml)
    - [pvc-azuredisk.yaml](deployment/azureDisk/pvc-azuredisk.yaml)
  - Apply the above yaml files
    - `k apply -f pv-azuredisk.yaml`
    - `k apply -f pvc-azuredisk.yaml`

> :warning: You need to be really careful on the size of nodes. Only certain type of nodes will support Azure Disk attach. `standard_d2a_v4 ` is not one of supported VMs.

### Understanding how external volumes are mounted

The problem here is quite obvious. You cannnot mount the persistent volume (Azure Disk in our case) to multiple hosts simultaneously in read/write mode. The only possible way is to set the Azure Disk into `read-only` mode.

Command to find out the attached volumes of a node: 
```
k get node <node-name> -o json | jq .status.volumeAttached
```


## Questions
- [X] If volumes share the pod lifecycle, will we lose the volumes when the pod restarted?
  - Section: 7.1.2: Understanding how volumes fit into pods
  - **Answer**: All volumes in a pod are created when the pod is set up - before any of its containers are started. They are torn down when the pod is shut down.
  - **Update**: Restart are for containers instead of pod. When we say restart, we mean the containers in the pod will be restarted. The pod will not be deleted. Thus, the volumes associated with the pod will persis until the pod got deleted.
- [X] When we created an volumes, did we create on the node file system? Check and verify use the `kubectl node-shell` command.
  - **Yes.** `emptyDir` will create a directory on the same host node, which host the pod.
- [ ] Why multiple pods in the same node can share the attached PV in read/write mode? When multiple pods write to the same file, will it cause consistency trouble? 

## ToDos
- [ ] Mount `azureFile` and `azureDisk` as a volume to a pod. Need concrete example.