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

Besides init containers, we can use **lifecycle hooks** to fulfill some requirements. Most cases, lifecycle hooks are used to execute some commands. **Also possible to execute to some scripts.** You can see the details in the example below.
- Post-start hooks: executed when the container starts
- Pre-stop hooks: executed shortly before the containerstops

`Init containers` are in pod level. However, lifecycle hooks are in container level. Following image show the details. 
![How the post-start and pre-stop hook fit into the container's lifecycle](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image010.png)


- Understand the command in post-start hook.
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: quote-poststart
  spec:
    containers:
    - name: nginx
      image: nginx:alpine
      ports:
      - name: http
        containerPort: 80
      lifecycle:
        postStart:
          exec:
            command:
            - sh
            - -c
            - |
              apk add fortune && \
              curl -O https://luksa.github.io/kiada/book-quotes.txt && \
              curl -O https://luksa.github.io/kiada/book-quotes.txt.dat && \
              fortune book-quotes.txt > /usr/share/nginx/html/quote
  ```
  `sh` calls the program `sh` as interpreter and the `-c` flag means execute the following command as interpreted by this program. [Reference](https://askubuntu.com/questions/831847/what-is-the-sh-c-command)


- Understanding how a post-start hook affects the container
  - **A container will stay in the `waiting` state due to `ContainerCreating` until the hook invocaton is completed.**
  - A container will keep restart if the post-start hook keeps failing (hook can't be executed or returns a non-zero exit code).
  >:exclamation:**Note: post-start hook acutally means postpone the start of the container.**

- Using an HTTP GET post-start hook
  - Example
  ```yaml
  lifecycle:
      postStart:
        httpGet:
          host: myservice.example.com
          port: 80
          path: /container-started
  ```

### Using pre-stop hooks to run a process before the container terminates

- Understanding how a pre-stop hook affects the container
  - When k8s need to terminate a container, it will send a `TERM` signal to the main process in the container.
  - If there is a pre-stop hook defined, then k8s will execute the hook **BEFORE** sending the `TERM` signal.

- Using a pre-stop lifecycle hook to shut down a container gracefully
  - Example
  ```yaml
  lifecycle:
      preStop:
        exec:
          command:
          - nginx
          - -s
          - quit
  ```

## Understanding the pod lifecycle

Pod lifecycle overview
- Initialization stage: init containers run
- Run stage: regular containers run
- Termination stage: pod's containers are terminated 

![The three stages of the pod's lifecycle](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image013.png)


### Understanding the initialization stage
Init containers run one by one, **the order is specified in the `initContainers` field in the pod's `spec` field.

- Image pull policy
  - Not Specified
    - Default policy is `Always` with `:latest` tag in the mage. Otherwide, the default policy will be `IfNotPresent`.
  - `Always`
    - Pull the image whenever the container is (re)started. **However, if the local image matches the one in the registry, image will not be downloaded, but the registry still needs to be contacted.**
  - `Never`
    - Image must present on the worker node beforehand.
  - `IfNotPresent`
    - Image is pulled only when it is not available on the worker node. This means the image will only be pulled the first time it's required.

> :warning: WARNING
  If the imagePullPolicy is set to `Always` and the image registry is offline, the container will **NOT** run even if the same image is already stored locally. **A registry that is unavailable may therefore prevent your application from (re)starting.**


- :exclamation: **Restarting failed init containers**
  If an init container terminates with an error and the pod’s restart policy is set to `Always` or `OnFailure`, the failed init container is restarted. If the policy is set to `Never`, the subsequent init containers and the pod’s regular containers are never started. The pod’s status is displayed as `Init:Error` indefinitely. You must then delete and recreate the pod object to restart the application.

- :exclamation: Re-executing the pod's init containers
  **Init containers must be idempotent.** Normally, init containers will only be executed once. Even though the main container got terminated, the pod's init containers will not be re-executed. However, if the entire pod got restarted, then the init containers might be executed again.

### Understanding the run stage
After all init containers started, k8s will create all the regular containers parallelly **in theory**. Ideally, containers' lifecycle should not be affected by each other. Following is the real situation: 

> :warning: **A container's post-start hook could block the creation of the subsequent container**
Kubelet creates and starts the containers synchronously according to the order defined in pod's `spec`. Exiting post-start hooks run asynchronously with corresponding main containers. However, the running post-start hook handler will block the start of the subsequent containers.

- Introducing the termination grace period
![A container's termination sequence](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image016.png)


### Understanding the termination stage
Containers will keep running until the deletion of the pod object.

- Deletion grace period
  The time interval is defined in `metadata.deletionGracePeriodSeconds` field. Default value is from `spec.terminationGracePeriodSeconds` field. **You can specify a different value in the `k delete` command. 
  - `k delete pod kiada-ssl --grace-period 10`

- How deletion grace period come in the picture
  ![The termination sequence inside a pod](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/06image017.png)

- Grace period sample
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kiada-ssl-shortgraceperiod
  spec:
    terminationGracePeriodSeconds: 5
    containers:
    ...
  ```

- Shutdown behavior of an application
  Sometimes, it takes a long time to shutdown an application. The cause could be that we didn't define a termination logic in the application. We need to make sure the process will listen to the `TERM` signal or process `SIGTERM`.


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
- [ ] Create an application that will process `SIGTERM`.
