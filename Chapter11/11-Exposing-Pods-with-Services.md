# Exposing Pods with Services

## 1. How Pods Communicate with Each Other in a Cluster

Every pod is isolated into its own network namespace and has its own IP address. However, all pods in the same cluster are connected by a single private network with a flat address space. Like many computers under a single router or switch, all pods can communicate with each other without NAT. This is the default behavior of Kubernetes. All network plugins work this way. Following is a diagram to illustrate this:

![Pods communicate via their own computer network](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/11image003.png)

[Document from Kubernetes](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)

## 2. Exposing Pods via Services

Pods are **ephemeral** and can be removed at anytime for many different reasons. When deploy an application, we normally have multiple pods running the same application. To expose the application either to the outside world or to other pods in the cluster, we need to create a **Service**. A service is a Kubernetes object that defines a logical set of pods and a policy to access them. Services are the abstraction that allows pods to be exposed to the outside world.

### 2.1 Introducing Services

A service is working as an abstraction layer for the backend pods. Services work as a load balancer between the backend pods and their clients (either in the cluster or outside the cluster.).

**Question 1:** If a service works a load balancer, does it do NAT for the traffic?
**Question 2:** When we have two service in the same K8s namespace, can they use the same port number?
  - **Answer:** Yes. They can.
  - **Reason:** They are in different network. How services isolated from each other?

- Create service
  - `k apply -f quote-svc.yaml`
  - `k expose pod quiz --name quiz`
- Get service
  - `k get svc -o wide`

### 2.2 Accessing Services from Inside the Cluster

- [ ] Interesting DNS issue: In 11.1.3, command `curl http://quiz` will not work. How to curl a service by its name in a node or in a pod?
- [ ] Run a curl pod to test the command in 11.1.3.
- [ ] Rebuild the image with curl installed.
- [ ] Create a cluster with public ip enabled.
- [ ] When I run `curl node-B-ip:node-port` from node A, the request could come back to node A. Does the request really go to node B?

### 2.3 Configuring the external traffic policy for a service 







