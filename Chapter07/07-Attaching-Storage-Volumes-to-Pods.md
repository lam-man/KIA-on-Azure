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
  - **Volumes mounted to a container cna be configured either as read/write or as read-only.**
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



## Questions
- [X] If volumes share the pod lifecycle, will we lose the volumes when the pod restarted?
  - Section: 7.1.2: Understanding how volumes fit into pods
  - **Answer**: All volumes in a pod are created when the pod is set up - before any of its containers are started. They are torn down when the pod is shut down.

## ToDos
- [ ] 