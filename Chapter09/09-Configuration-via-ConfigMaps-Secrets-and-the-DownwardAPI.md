# Configuration via ConfigMaps, Secrets, and the Downward API

The focus of this chapter is to exposing the pod information (or other information) to the applications running in a pod.

## Setting the command, arguments, and environment variables


### Commands, arguments with Dockerfile
In many cases, we need to configure the containerized application. Configurations can be command-line arguments or environment arguments. A common way is using docker file. Following is an example of kiada: 
```dockerfile
FROM node:12
COPY app.js /app.js
COPY html/ /html

ENV INITIAL_STATUS_MESSAGE="This is the default status message"

ENTRYPOINT ["node", "app.js"]
CMD ["--listen-port", "8080"]
```

In the above dockerfile, we use `ENV` and `CMD` to configure the application. 
- The configuration is hardcoded into the dockerfile. Every time we change the port number nad initial status message, we need to rebuild the image.
- Another reminder is sensitive information, for example security credentials and encryption keys, cannot be included in docker image, since people who has access to it can easily extract them.
- We can also save the sensitive information into a persistent volume like `emptyDir` and fetch them using an init container.
- ****

### Override dockerfile commands and arguments in pod manifest

In K8s, we can use `command` and `args` to override `ENTRYPOINT` and `CMD` in dockerfile.

Following is the example of overriding: 
![Overriding the command and arguments in the pod manifest](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/09image002.png)

- Command sample in a pod yaml file
```yaml
kind: Pod
spec:
  containers
  - name: kiada
    image: luksa/kiada:0.4
    command: ["node", "--cpu-prof", "--hea-prof", "app.js"]
```

- When we have many more arguments, we can use the following format to increase the readability: 
```yaml
command:
    - node
    - --cpu-prof
    - --heap-prof
    - app.js
```

- Tip
> Normally, yaml parser will interpret all the values as string. To avoid this, we need to quote certain values. For example numeric values and boolean values. `1234`, `true`, `false`, `yes`, `no`, `on`, `off`, `y`, `n`, `t`, `f`, `null`.

### Setting environment variables in a container using pod manifest

> NOTE: We are not able to set a global set of env variables for the entire pod and have them inherited by all its containers.

- Set env variables in pod manifest
  ```yaml
  kind: Pod
  metadata:
    name: kiada
  spec:
    containers:
    - name: kiada
      image: luksa/kiada:0.4
      env:
      - name: POD_NAME
        value: kiada
      - name: INITIAL_STATUS_MESSAGE
        value: This status message is set in the pod spec.
      ...
  ```

- Command to check the env var in a cluster
  `k exec kiada -- env`

- Using variable references in env variable values
  ```yaml
  env:
  - name: POD_NAME
    value: kiada
  - name: INITIAL_STATUS_MESSAGE
    value: My name is $(POD_NAME). I run NodeJS version $(NODE_VERSION).
  ```

- :warning: Drawbacks of variable referencing
  Variable referencing in k8s `$(VAR_NAME)` only works for variables defined in the same manifest. **The referenced variable must be defined before the variable that references it.** If a referenced variable cannot be resolved, the reference string will remain unchanged.

- Using variable references in the `command` and `arguments`
  In the `command` and `args` section, you can also use variable reference to reference the variables in `env` section.
  ```yaml
  spec:
    containers:
    - name: kiada
      image: luksa/kiada:0.4
      args:
      - --listen-port
      - $(LISTEN_PORT)
      env:
      - name: LISTEN_PORT
        value: "8080"
  ```

- Referencing to env variables that aren't in the manifest
  As the previous section mentioned, variable referencing only works for variables defined in the pod manifest. We can bypass this using with `shell` command. 
  ```yaml
  containers:
  - name: main
    image: alpine
    command:
    - sh
    - -c
    - 'echo "Hostname is $HOSTNAME."; sleep infinity'
  ```

## Using a config map to decouple configuration from the pod

With `command`, `args` and `env`, we can hardcode configuration into the pod manifest. It is not an ideal way, as we need to keep multiple version of pod manifest (for the same pod) for different environment like: staging, dev, and prod.

### What is ConfigMaps

**A ConfigMap is a Kubernetes API object that simply contains a list of key/value pairs.** Pods can reference one or more of these key/value entries in the config map. A pod can refer to multiple config maps, and multiple pods can use the same config map.

- How ConfigMap object works with k8s? 
  Typically, **applications will not read the ConfigMap objects via the k8s REST API.** ConfigMap have 2 ways to associate key/value pairs with k8s: 
  - Key/Value pairs in the config map are passed to contianers as env variables.
  - Mounted as files in the containers' filesystem via a `configMap` volume, as shown in the following picture.
  ![Pods use config maps through environment variables and configMap volumes](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/09image004.png)

- Benefits
  - Without taking the way applications consuming `configMap`, a `configMap` object keep the application and configuration separated.
  - Although we no longer need to maintain multiple pod manifests for the same pod, **we need to maintain several `configMap` yaml files.** 
  ![Deploying the same pod manifest and different config map manifests in different environments](https://drek4537l1klr.cloudfront.net/luksa3/v-14/Figures/09image005.png)

### Creating a ConfigMap object

We have different ways to create a `configMap` object in k8s. Following are the common ways: 
- `kubectl` command
  ```bash
  k create configmap kiada-config --from-literal status-message="This status message is set in the kiada-config config map"
  ```
- Create yaml file and use `k apply -f`
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: kiada-config
  data:
    status-message: This status message is set in the kiada-config config map
  ```
- Command Command
  - Get `configMap`
    `k get cm` --> `cm` short for `configmap` or `configmaps`
  - Display all the key value pairs in a config map
    `k get cm <cm-name> -o json | jq .data`
  - Display the value of a given key
    `k get cm <cm-name> -o json | jq '.data["<key-name>"]'`

### Injecting config map values as environment variables

- Injecting a single config map entry using `valueFrom` in yaml
  ```yaml
  kind: Pod
  ...
  spec:
    containers:
    - name: kiada
      env:
      - name: INITIAL_STATUS_MESSAGE
        valueFrom:
          configMapKeyRef:
            name: kiada-config
            key: status-message
            optional: true
    volumeMounts:
    ...
  ```
  - Check the env environment of a pod
    `k exec kiada -- env`
  
- Marking a reference optional using `optional` in pod manifest
  ```yaml
  ...
  - env:
    - name: INITIAL_STATUS_MESSAGE
      valueFrom:
        configMapKeyRef:
          key: status-message
          name: kiada-config
          optional: true
  ```
  > :exclamation: **NOTE:** With `optional` set to `true`, a pod will executed even if the config map or key is missing.

- Injecting the entire config map using `envFrom`
  Add entries one by one int `env` section is tedious. To avoid specifying each entry, we can use `envFrom` to load the entire config map.
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kiada
  spec:
    containers:
    - name: kiada
      image: luksa/kiada:0.4
      env:
      - name: POD_NAME
        value: kiada
      envFrom:
      - configMapRef:
          name: kiada-config
          optional: true
      ports:
      - name: http
        containerPort: 8080
  ```
  - :exclamation: **Important Notes:** 
    - The `optional: true` here is special, since the default is `false`. Defaultly, the pod will fail to start ifthe configmap doesn't exist.
    - You can have **mutiple** configMap in `envFrom`
    - You can `combine` the `env` and `envFrom` fields. Above pod manifest is an example.
    - Prefixing keys
      You can set an optional `prefix` for each config in the `envFrom` field. 

### Injecting config map entries into containers as files

Usually, we use environment variables to inject single-line values and multi-line values will be passed as files. For `configMap` that contains larger blocks of data, we mount the `configMap` as a volume.

With `configMap` vlolume, we can the entries in it available as individual files. Then the process running in the container gets the entry's value by reading the contents of the file. **This is the popular way to pass large config file to a container.** Absolutely, we can combine it with `env` or `envFrom`.

> :exclamation: **NOTE on ConfigMap Size:** etcd (underlying data store for k8s api objects) decided the amount of information that can fit in a `configMap`. For now, the max size is 1 MB

- Create a `configMap` with file as value using command line
  ```bash
  k create configmap kiada-envoy-config \
    --from-file=envoy.yaml \
    --from-file=dummy.bin \
    --dry-run=client -o yaml > cm.kiada-envoy-config.yaml
  ```

  - `--dry-run`: tell the command do not create the object in k8s API server, but only generates the object definition. 
    - More on `--dry-run`: 
      > Value after `=` Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without sending it. If server strategy, submit server-side request without persisting the resource.
  
  - A sample result of the above command line can be found at
    - [cm.kiada-envoy-config.yaml](./deployment/cm.kiada-envoy-config.yaml)
    - Explanation: 
      - binary file will be in `binaryData` filed and will be encoded in Base64 format. 
      - yaml file will be in `data` field.

  - :warning: NO TRAILING WHITESPACE IN THE VALUE YAML FILE
    If you have trailing whitespace (whitespace at line end) in the yaml file you want to put as an entry in a configMap, the format of the configMap will be messed up.

- Using a configMap volume in a pod
  - Sample pod manifest
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kiada-ssl
    spec:
      volumes:
      - name: envoy-config
        configMap:
          name: kiada-envoy-config
      ...
      containers:
      ...
      - name: envoy
        image: luksa/kiada-ssl-proxy:0.1
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
      ...
    ```

  - Marking a configMap volume as optional
    When a container contains an environment varialbe that references an not-existed config map, **the container will be prevented from starting (if the env var is not optional). However, other containers will NOT be affected.** 

    :exclamation: **This is not the case for config map as a volume.** A container with missing configMap (as a volume) will prevent all the containers from stating until the configMap got created. This is becuase all the pod's volumes must be set up before the pod's container can be started.

    - > **NOTE on setting configMap volume as optional:** Config Map volume can be set as optional. With `optional: true`, the missing configMap volume will not affect the start of containers. (With optional, the configMap volume will not be created.)

  - Projecting only specific config map entries
    There are cases we want to associate certain files in a configMap. This can be achieved by using `items` in a configMap volume. Only entries in the `items` list will be included in the volume. 
    ```yaml
    volumes:
      - name: envoy-config
        configMap:
          name: kiada-envoy-config
          items:
          - key: envoy.yaml
            path: envoy.yaml
     ```

  - Setting file permissions in a configMap volume
    - Default permission: 
      - Regular: `rw-r--r--`
      - Octal: `0644`
    - Notes
      - Set the `mode` field next to the item's `key` and `path`
      - :warning:You need to use the octal number to indicate the permissions. **DONOT forget the leading 0.**
        - Example: `defaultMode: 0740`
      

### Updating and deleting config maps

- In-place editing of API objects using `kubectl edit`
  - Example: 
    `k edit configmap kiada-envoy-config`
- What happens when you modify a config map 
  - `configMap` volume
    The `configMap` volume will be automatically updated when a config map got updated. **However, there will be a minute delay for the `configMap` volume to be udpated.**
  - Environment variables 
    The env variables that reference the config map will not be updated until the containers restarted automatically or manually.
- Understanding the consequences of updating a config map
  As we all know, containers are immutable in k8s. Naturally, we would ask that should we make their configration immutable?
  When we update the `configMap`, containers in the cluster will not be updated at the same time. Especially, k8s created to maintain the high availability of services. This means at some point, some pods in the cluster will be different with the rest of the cluster.
- Preventing a config map from being updated
  - Example
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-immutable-configmap
    data:
      mykey: myvalue
      another-key: another-value
    immutable: true
    ```
  - Benefit
    Immutable config maps will help improve the performance of a k8s cluster, as users are not allowed to update config maps and `kubelets` don't have to be notified of changes to the ConfigMap object.
- Deleting a config map
  - Command
    `k delete`

### Understanding how configMap volumes work


## Using Secrets to pass sensitive data to containers


## Reference Reading
- [kubectl-commands](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- 


## Questions

## ToDos
- [ ] Figure out how Helm works with `configMap`. Concrete examples for the following task:
  - [ ] How to create a Helm chart for a service?
  - [ ] How to reference key values in a `configMap` from Helm chart?
- [ ] Concrete example on multiple `configMap` file and prefixing keys for each `configMap`.
- [ ] What are the differences between `configMap` and `secrets`?
  - [ ] Read [Kubernetes Secrets Are Not Really Secret!](https://auth0.com/blog/kubernetes-secrets-management/)