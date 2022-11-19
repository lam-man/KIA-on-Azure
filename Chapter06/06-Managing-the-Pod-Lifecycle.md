# Managing the Pod Lifecycle

## Understand the pod's status

### Pod's phase
The following diagram shows the details of a pod's lifecycle. 
![The phases of a Kubernetes pod](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image002.png)

- **Display a pod's phase**
You can get the a pod's phase trhough the yaml file.
  - Commands
    - `k get po kiada -o yaml | grep phase`
    - `k get po kiada -o json | jq .status.phase`
    - `k describe po kiada`
      - You can see the status of the pod.
Command `k get pods` works only when the pod is is in healthy status.

### Understanding pod conditions 

- List of pod conditions
  - PodScheduled: Indicates whether or not the pod has been scheduled to a node.
  - Initialized: The pod's init containers have all completed successfully.
  - ContainersReady: All containers in the pod indicate that they are ready. This is a necessary but not sufficient condition for the entire pod to be ready.
  - Ready: The pod is ready to provide services to its clients. 

- How to get the conditions?
  - `k get po kiada -o json | jq .status.conditions`
  - `k describe po kiada`

### Understanding the container status 

**Container status actually provides more information for cluster admin, as it is the client in the pod and the culprit of pod issues.**
- Container status sample: 
```yaml
containerStatuses:
  - containerID: containerd://0438d79e6fc85d56e8db532ded6977819e5162bbeb185b58b3a18e6cf3c77c07
    image: docker.io/luksa/kiada-ssl-proxy:0.1
    imageID: docker.io/luksa/kiada-ssl-proxy@sha256:ee9fc6cfe26a53c53433fdb7ce0d49c5e1bffb889adf4d7b8783ae9f273ecfe7
    lastState: {}
    name: envoy
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-11-19T06:06:56Z"
  - containerID: containerd://99c7390e6caed24e87e40b581b68d832b64f80778857213e652741389c9c5f48
    image: docker.io/luksa/kiada:0.2
    imageID: docker.io/luksa/kiada@sha256:901847a775d8ab631df844c40555a8cbfd2c5ab2b9d9a0913d5216b07db7b1d9
    lastState: {}
    name: kiada
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-11-19T06:06:51Z"
```

- Fields details:
  - `state`: current state of the container
  - `lastState`: state of the previous container after it has terminated. 
  - `restartCount`: how many times it restarts.

- Status illustration
![The possible states of a container](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image004.png)

- **Displaying the status of the pods's containers**
  - Commands
    - `k describe po kiada`
    - `k get po kiada -o json | jq .status.containerStatuses`
    - `k get po kiada-init -o json | jq .status`
