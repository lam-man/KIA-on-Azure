# Running Worloads in Pods

## Understanding Pods

Pod is the smallest deployable unit in Kubernetes. A pod will have one or more containers. Containers in the same pod will run on the same worker node -- a single pod instance never spans multiple ndoes. 

![All containers of a pod run on the same node.](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/05image003.png)

### Understanding why we need pods 

- Why one container shouldn't contain multiple process?
  - Difficult to manage
    - With multiple process, the log will be interwined and hard to read.
  - Although root process can spawn child processes, restarts only happen when root process dies.

- How a pod combines multiple containers?
  - We need something to group the related processes together, although they are running in multiple contianers.
  - Inside a pod, containers share certain resources like network, UTS namespace, IPC namespace (yes, containers in the same pod can communicate using IPC), and file system (mounted to the pod). Details in Chapter 7.

![Containers in a pod share the same network interfaces](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/05image004.png)


## ToDos
- [ ] Deep dive into container runtime. Watch [Introduction and Deep Dive into containerd](https://www.youtube.com/watch?v=HFEZq2YddPU)
- [ ] [Docker Deep Dive](https://app.pluralsight.com/library/courses/docker-deep-dive-update/table-of-contents)
- [ ] What are pods and containers? Read [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)