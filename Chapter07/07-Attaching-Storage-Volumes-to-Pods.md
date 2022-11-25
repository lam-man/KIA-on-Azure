# Attaching Storage Volumes to Pods

## Introducing volumes
When a container is terminated and restarted, all changed files are lost. This is because k8s simply replaced the container instead of restart. Thus, we need to find a way to persist changes when need.this can be achieved by adding a volume to the pod and mounting it into the container.

- Mounting
  Attaching a file system of some storage device or volume into a specific location in the operating system's file tree. 
  ![Mounting a filesystem into the file tree](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/07image002.png)



