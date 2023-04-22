# Managing Pods with Deployments

## 1. Introducing Deployments

Workloads are usually deployed through a Deployment. However, deployment is not the one to manage the pods. It is the underneath ReplicaSet that manages the pods. The deployment is used to manage the ReplicaSet.

![Relationship between Deployment, ReplicaSet and Pods](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/14_img_0001.png)

### 1.1 Creating a Deployment
Instead of creating a deployment by writing the yaml file, we can use the `kubectl create deployment` command to create a deployment.
- Creating 
  - `k create deployment kiada --image=luksa/kiada:0.1`
  - `k create deployment kiada --image=luksa/kiada:0.1 --dry-run -o yaml > kiada-deployment.yaml`
- Inspecting
  - `k get deploy kiada`
  - `k describe deploy kiada`
- Checking rollout status
  - `k rollout status deploy kiada`
  - `k get pods -l app=kiada,rel=stable`
- Deployment naming
  When you create a deployment, the deployment name will be used as the prefix for the replicaset name. There will also be a suffix from the label `pod-template-hash`. We should be able to find the `pod-template-hash` using `k describe rs kiada`.
- Will the exiting pods be adopted by the replicaset created by deployment?
  - No, the existing pods will not be adopted by the replicaset as there is one more label `pod-template-hash` in the replicaset selector. The existing pods don't have this label.
  ![Label selectors in the Deployment and ReplicaSet, and the labels in the Pods](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/14_img_0002.png)
  - `k get po -l app=kiada` # Show pods 
  - `k get po --show-labels| grep kiada` # Show pods with labels
  - `k edit rs kiada-<suffix>` # Show the replicaset selector

### 1.2 Scaling a Deployment
Scaling a deployment is pretty much the same as scaling a replicaset. The actual work is done by the replicaset controller.
- Command
  - `k scale deploy kiada --replicas 5` # Scale up
  - `k scale deploy kiada --replicas 2` # Scale down
  - `k get po -w` # Watch the pods creation and deletion
- All your actions on the deployment will be recorded in the events.
  - `k describe deploy kiada` # Check the events at the bottom
  - `k get events --sort-by=.metadata.creationTimestamp` # Not very useful
- Scale replicaset that owned by a deployment wil not work. The scale up work done by rs controller will be revereted by the deployment controller. No matter what changes you did to the underlying rs, the deployment controller will always try to make the rs match the deployment spec.
  - `k scale rs kiada-<suffix> --replicas 5` # This will show scale work is done
  - `k get rs -w` # This will show the replicaset is scaled back to 2 (changes revered)

### 1.3 Deleting a Deployment
- How to preserve the underlying replicaset and pods when deleting a deployment?
  - `k delete deploy kiada --cascade=orphan` # Delete the deployment but keep the replicaset and pods
  - `k get rs` # Check the replicaset
  - `k get po` # Check the pods
- When you create deployment again, it will try to adopt the existing replicas and pods.


## 2. Updating a Deployment
The benefit to use a deployment is in updating pods. Replicaset doesn't provide those convinient operations.

- Update strategies
  - Currently, there are two update strategies: `RollingUpdate` and `Recreate`. The default strategy is `RollingUpdate`.
  - ![The difference between the Recreate and the RolliongUpdate strategies](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/14_img_0004.png)

- Image update
  - Deployment provides a declarative way to update the image of the pods.
  - `k set image deployment kiada kiada=luksa/kiada:0.6` # Update the image
  - `k get rs -w` # Watch the replicaset creation and deletion
  - `kubectl patch deploy kiada --patch '{"spec": {"template": {"metadata": {"labels": {"ver": "0.6"}}, "spec": {"containers": [{"name": "kiada", "image": "luksa/kiada:0.6"}]}}}}'` # Update the image using patch
  - More readable command
    ```bash
    kubectl patch deploy kiada --patch '
    spec:
      template:
        metadata:
          labels:
            ver: "0.6"
        spec:
          containers:
          - name: kiada
            image: luksa/kiada:0.6'
    ```
  - The previous replicaset will be kept and scale down to 0
    - `k get rs -L ver` # Show the replicaset version
    ![Updating a Deployment](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/14_img_0005.png)

### 2.1 Rolling Update
With the `Recreate` strategy, all pods will be deleted simultaneously and then recreated. Down time will be introduced during the update. Solution to resolve the down time issue is to use the `RollingUpdate` strategy. The `RollingUpdate` strategy will update the pods one by one or in a pace defined by the customers. (75 - 125).

- How it works?
  - ![What happens with the ReplicaSets, Pods, and the Service during a rolling update](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/14_img_0006.png)
- Configuration
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: kiada
  spec:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 0
        maxUnavailable: 1
    minReadySeconds: 10
    replicas: 3
    selector:
    ...
  ```
- Show the rollout update details
  - `k apply -f deploy.kiada.0.7.rollingUpdate.yaml`
  - `k get rs -w` # Watch the replicaset creation and deletion
  - `k rollout status deploy kiada` # Show the rollout status
  - `k get pod -l pod-template-hash=<value> -w` # Show the pods

- Show the rollout update using a test client
  ```bash
  kubectl run -it --rm --restart=Never kiada-client --image curlimages/curl -- sh -c \
  'while true; do curl -s http://kiada | grep "Request processed by"; done'
  ```

### 2.3 Configuring how many Pods are replaced at a time
- `maxSurge` and `maxUnavailable` are used to control how many pods are replaced at a time.

- `maxSurge` is the maximum number of pods that can be created over the desired number of pods. It can be a number or a percentage. The default value is 25%.
  - `maxSurge: 0` # No new pods will be created
  - `maxSurge: 1` # One new pod will be created
  - `maxSurge: 50%` # 50% of the desired number of pods will be created
- `maxUnavailable` is the maximum number of pods that can be unavailable during the update. It can be a number or a percentage. The default value is 25%.
  - `maxUnavailable: 0` # No pods will be deleted
  - `maxUnavailable: 1` # One pod will be deleted
  - `maxUnavailable: 50%` # 50% of the desired number of pods will be deleted

- Test
  - `k apply -f deploy.kiada.0.6.recreate.yaml`
  - Update the replicas to 8
  - Set `maxSurge: 0` and `maxUnavailable: 2` in `deploy.kiada.0.7.rollingUpdate.yaml`
  - `k get po --show-labels`
  - `k get po -l pod-template-hash=5c96bc9585 -w` # Watch the pods
  - `k apply -f deploy.kiada.0.7.rollingUpdate.yaml`

### 2.4 Pausing the rollout process
I personally don't think this is a good idea. It would be easier to just test by change the image version for the container you want to update. However, I am not sure this is a good idea test a pod owned by the deployment.
- `k rollout pause deploy kiada` # Pause the rollout

### 2.5 Updating to a faulty version
Instead of using `k rollout pause deploy kiada` to pause the rollout and check if the new version is good, we can make k8s to do the work automatically.

We can use the `minReadySeconds` to define how long a pod should be operated well before it is considered as ready. If the pod is not ready for the defined time, the rollout will be paused.

> Note: `minReadySeconds` will be reset when the pod is restarted.

- `k apply -f deploy.kiada.0.8.minReadySeconds60.yaml`
- `k describe deploy kiada` # Check the `minReadySeconds` value
- Check the conditions for the deploy.

### Rolling back a Deployment
With deployment, we can rollback to a previous version easily with the following command.

- `k rollout undo deploy kiada` # Rollback to the previous version
- `k applyf -f deploy.kiada.0.7.rollingUpdate.yaml` # Update the image to 0.7
- Display rollout history for a deployment
  `k rollout history deploy kiada`



## TODO
- [ ] Play with session affinity in k8s.
 


