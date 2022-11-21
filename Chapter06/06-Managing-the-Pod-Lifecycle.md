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

## Keeping containers healthy
Containers running in a pod could die easily. What we want to learn in this section is keep the containers healthy and running.

### Understanding container auto-restart
Auto healing is coming with k8s. As a pod is alive, kubelet will keep the container running. If the main process crashes, kubelet will restart the container.

- How to kill a envoy proxy container? 
  - Visit `http://localhost:9901` and click the `quitquitquit` button. 
  - `curl -X POST http://localhost:9901/quitquitquit`

- What happened when a container got killed?
  K8s will not restart any container. Instead, it will kill the container and create a new one. With the killing, following are the effects. 
  - We will lose the contianer's filesystem, which the main process writes to. (In next chapter, we will learn how to add a storage volume to a pod.)
  - If init containers are defined in the pod and one of the pod's regular containers is restarted, the init containers are not executed again.

- Configuring the pod's restart policy
  `restartPolicy` is configurable in Kubernetes. Following are the restart policies in k8s. 
  - **Always**: Container is restarted regardless of the exit code the process in the container terminates with. **This is the default restart policy.**
  - **OnFailure**: The container is restarted only if the process terminates with a non-zero exit code, which by convention indicates failure.
  - **Never**: The container is never restarted - not even when it fails.

- **NOTE**: The restart policy is configured at the pod level and applies to all its containers. Configuration for each container separately is not possible.

- Time delay before a container is restarted
  | Restart Count | Deplay Time |
  |---------------|-------------|
  |1              |0 (immediately)|
  |2              |10s          |
  |3              |20s          |
  |4              |40s          |
  |5              |80s          |
  |...            |5 mins       |

- Command to get the container status
  - `k get po kiada-ssl -o json | jq .status.containerStatuses`

### Checking the container's health using liveness probes

A process can stay in deadlock status without stopping. Thus, the container will not be restarted. In order to restart the unhealthy container forcely, we want to use liveness probes.

> :exclamation: **Note:  Liveness probes can only be used in the pod's regular containers. They can't be defined in init containers.**

- Types of liveness probes
  - **HTTP GET**: Send http get request to container's IP address and expect success response in time. Otherwise, the container is considered unhealthy.
  - **TCP Socket**: Establish a TCP connection to a specific port of the container successfully. Otherwise, mark container as unhealthy.
  - **Exec**: Run a command inside the container.

- Sample of liveness probe
  ```yaml
  livenessProbe:
    httpGet:
      path: /ready
      port: admin
    initialDelaySeconds: 10
    periodSeconds: 5
    timeoutSeconds: 2
    failureThreshold: 3
  ```
- Liveness probe fields explanation
  ![The configuration and operation of a liveness probe](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image007.png)

- Liveness probe verification through log
  - Commands
    - `k logs kiada-liveness -c kiada -f`
    - `k exec kiada-liveness -c envoy -- tail -f /tmp/envoy.admin.log`

- :exclamation: **Error exit code note**:
  Exit code 128+n indicates that the process exited due to external signal n. Exit code 137 is `128+9`, where `9` represents the `KILL` signal. Exit code `143` is `128 + 15`, where `15` is the `TERM` signal, which indicates the container runs a shell that has terminated gracefully.

### Using a startup probe when an application is slow to start 
If an application lake minutes to start, then short time liveness probe could prevent the container from reaching expected status. Although we can achieve longer delay by configuring `initialDelaySeconds`, `periodSeconds` and `failureThreshold`. 

- Start up probe sample
  ```yaml
  ...
  containers:
  - name: kiada
    image: luksa/kiada:0.1
    ports:
    - name: http
      containerPort: 8080
    startupProbe:
      httpGet:
        path: /
        port: http
      periodSeconds: 10
      failureThreshold:  12
    livenessProbe:
      httpGet:
        path: /
        port: http
      periodSeconds: 5
      failureThreshold: 2
  ```

- Failure is normal for a startup probe
  A failure just means the application is ready yet.

- Creating effective liveness probe handlers
  - A liveness probe is needed, otherwise your application will remain in unhealthy status and need a manual restart.
  - Expose a specific health-check endpoint and make sure no authentication is needed for the entpoint.
  - A liveness probe should only check the status of the target application. No dependent applications should be checked.
  - Keeping probes light. Use less CPU and memory.

## Executing actions at container start-up and shutdown



## :question: Questions
- Can we configure `initialDelaySeconds` only for applications that are slow to start? 
  - In this case, we will wait for enough time at the begining. The restart will not be affected as `periodSeconds` and `failureThreshold` will be small.
  - **Answer**: I think we cannot do this. 
    - We don't know how long exectly we need to start an application. Thus `initialDeplaySeconds` may not be able to cover the startup time precisely.
    - During the application start interval, we want to periodically check whether the applicaton is ready. Then we need to set `periodSeconds` and `failureThreshold` to check with a long interval, which will affect the restart.






## ToDos
- [ ] How to define and find the restart policy of a pod? 
  - [ ] Create concrete examples.
- [ ] Create an application with liveness probe port configured.