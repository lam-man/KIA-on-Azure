# Replicating Pods with ReplicaSets

## 1. Inroducing ReplicaSets
ReplicaSets select and manage pods using labels. **Labels for a replicaset and the pod template must match, otherwise k8s will not create the pods.**

### 1.1 Insepecting a ReplicaSet and its Pods
- Get the details of a replicaset
  - `kubectl get rs kiada`
  - Where does k8s store the information for `replicas`, `fullyLabeledReplicas`, `readyReplicas` and `availableReplicas`?
  - Guess: store in the `etcd` database and returned by the `kube-apiserver`.

- When you create a replicaset, existing pods with the same labels will be adopted and managed by the replicaset.
  - If I delete the rs, will the pre-created pods be deleted?
    - It will be deleted all at once. **K8s has a garbage collector. The garbage collector will delete the pods after the owner rs is deleted.**
    - [ ] Check out source code of the garbage collector.

- The name of the replicaset will be used as the prefix for the pods name. 
  - `k edit po <kiada-pod-name>`
  - Check field `metadata.generateName`

### 1.3 Pod ownership
In the pod yaml file, there is a field `metadata.ownerReferences` which is used to record the owner of the pod.
- `kubectl get po <pod-name> -o yaml`
```yaml
 ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: kiada
    uid: 9d659338-7e2b-4c8a-ae8d-81d7d2d97964
```

## 2. Updating a ReplicaSet
- Scaling a ReplicaSet up and down
  - `k scale rs kiada --replicas=6`
  - `k get rs kiada -w`
- There is an order to delete the pods: 
  - Pods that aren’t yet assigned to a node.
  - Pods whose phase is unknown.
  - Pods that aren’t ready.
  - Pods that have a lower deletion cost.
  - Pods that are collocated with a greater number of related replicas.
  - Pods that have been ready for a shorter time.
  - Pods with a greater number of container restarts.
  - Pods that were created later than the other Pods.
- Pod deletion can be customized with annotation `controller.kubernetes.io/pod-deletion-cost`.

- Scale down to zero
  - Scale down to zero will delete all pods but not the replicaset. This could be used when you want to refresh all pods.

## 3. ReplicaSet Controller

### 3.1 Introducing the reconcilliation control loop

## 4. Delete a ReplicaSet
- Delete a replicaset without deleting all the pods managed by the replicaset.
  - `k delete rs kiada --cascade=orphan`
  - `k get rs kiada`

## Lab
The following are commands to reproduce the experiment.
```bash
k apply -f SETUP -R  # Prepare the environment
k apply -f rs.kiada.yaml  # Create the replicaset
k get rs -w  # Watch the replicaset craetion
k get po -l app=kiada,rel=stable # Check the pods created by the replicaset
k delete rs kiada  # Delete the replicaset, all pods will be deleted
k get po # all kiada pods will be deleted
k apply -f rs.kiada.yaml  # Create the replicaset again
k edit pod <pod-name> # show the pod owner in metadata.ownerReferences
k scale rs kiada --replicas=7 # scale up the replicaset
k get rs kiada # watch the replicaset
k scale rs kiada --replicas=4 # scale down the replicaset
k get rs kiada # watch the replicaset
k get pods -l app=kiada,rel=stable # check the pods to see if any pods got deleted

```
