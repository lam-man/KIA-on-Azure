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


## Questions

## ToDos
- [ ] Figure out how Helm works with `configMap`. Concrete examples for the following task:
  - [ ] How to create a Helm chart for a service?
  - [ ] How to reference key values in a `configMap` from Helm chart?