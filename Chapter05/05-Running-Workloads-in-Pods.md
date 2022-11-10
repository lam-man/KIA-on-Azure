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

## Interacting with the application and the pod

> **Note on Ports**: 
Port declaration is purely informative. Omission of port declaration in pod definition will not affect the connection of ports. If we know that a container is listening on certain ports, we can connect to it even without declare it in the pod spec. 

### How to `curl` your service in a pod? 

Normally, we talk to the application in pod through the service in cluster. However, for testing purpose, we may want to use the fatest way -- `curl`. To do that, we can choose the following methods: 
- Connect to your node
  - To ssh into a node in a K8s cluster, we can use [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell). It is a tool helps to ssh into node. 
  - Commands 
    - `k node-shell <node-name>`
    - `curl 10.244.2.4:8080` Change IP to connect your node.
- Start a new pod run curl
  - You can read more from [How to run curl in Kubernetes](https://www.tutorialworks.com/kubernetes-curl/)
  - Commands
    - `k run mycurlpod --image=curlimages/curl -i --tty -- sh`
    - Kubernetes will pull image from Docker hub.
    - `k run --image=curlimages/curl -it --restart=Never --rm client-pod curl 10.244.2.4:8080` Delete immediately after using it.
- Connect to your pod
  - We can also connect to a pod as well. This may not be correct, as we can curl with `localhost`.
  - Commands
    - `k exec --stdin --tty <pod-name> -- /bin/bash`
- Connecting to pods via kubectl port forwarding 
  - Commands
    - `k port-forward kiada 8080:<Port-You-Want-in-Local>`
    - `k port-forward kiada 8080:9090`

### Viewing application logs
- Commands 
  - `k logs <pod-name>` Ex: `k logs kiada`
  - `k logs <pod-name> -f`
  - `k logs kiada --timestamps=true`
  - Displaying recent logs 
    - `k logs kiada --since=2m`
    - `k logs kiada --since-time=2020-02-01T09:50:00Z`
  - Displaying last number of lines
    - `k logs kiada --tail=5` or `k logs kiada --tail 5`

- **Understanding the availability of the pod's logs**
  - Each container has its own log file in `/var/log/containers`. If the container is restarted, a new log file will be created and written into. If a user is using `k logs -f`, then the command will be terminated. 
  - Command `k logs` only display the logs for current container. You need `k logs --previous` or `k logs -p` to view the logs from previous container.


### Copying files to and from containers


## Questions
- [ ] Does a pod have an OS?
  > In 5.1.2: You never need to combine multiple applications in a single pod, as pods have almost no resource overhead.
- [ ] How AKS figure out where to pull the specified image?
  > Answer: Docker Hub is the default registry for AKS. If you didn't specify a registry, then default one will be used.
- [ ] How to set a default container registry for my AKS cluster?
- [ ] How `k port-forward` works?


## ToDos
- [ ] Deep dive into container runtime. Watch [Introduction and Deep Dive into containerd](https://www.youtube.com/watch?v=HFEZq2YddPU)
- [ ] [Docker Deep Dive](https://app.pluralsight.com/library/courses/docker-deep-dive-update/table-of-contents)
- [ ] What are pods and containers? Read [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)