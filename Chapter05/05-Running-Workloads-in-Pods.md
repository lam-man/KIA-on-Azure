# Running Worloads in Pods

## Understanding Pods

Pod is the smallest deployable unit and scaling unit in Kubernetes. A pod will have one or more containers. Containers in the same pod will run on the same worker node -- a single pod instance never spans multiple ndoes. 

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

- Why we shouldn't deploy multiple applications to the same pod? 
  ![Splitting an application stack into pods](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/05image005.png)

- How to decide whether to split containers into multiple pods? 
  - Do I need to manage them as a single unit? 
  - Do they form a unified whole instead of being independent components? 
  - **Do they have to be scaled together?**
  - **Can a signle node meet their combined resource needs?**

## Create pods from YAML or JSON

### Quickly checking the status of a pod
- `k get po kiada`
  ```md
  NAME    READY   STATUS    RESTARTS   AGE
  kiada   1/1     Running   0          6m34s
  ```
- `k get po kiada -o wide` 
  ```md
  NAME    READY   STATUS    RESTARTS   AGE     IP            NODE                                NOMINATED NODE   READINESS GATES
  kiada   1/1     Running   0          3m20s   10.244.0.17   aks-nodepool1-39758779-vmss000000   <none>           <none>
  ```
- `k describe po kiada`
  Show you how the details of the pod, which includes the events.(Events will be cleaned after an hour.)

## Questions
- [ ] Does a pod have an OS?
  > In 5.1.2: You never need to combine multiple applications in a single pod, as pods have almost no resource overhead.
- [ ] 

## ToDos
- [ ] Deep dive into container runtime. Watch [Introduction and Deep Dive into containerd](https://www.youtube.com/watch?v=HFEZq2YddPU)
- [ ] [Docker Deep Dive](https://app.pluralsight.com/library/courses/docker-deep-dive-update/table-of-contents)
- [ ] What are pods and containers? Read [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)