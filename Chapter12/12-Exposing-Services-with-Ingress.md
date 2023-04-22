# Exposing Services with Ingress

## 1. Introducing Ingresses

- The word `ingress`: 
  The act of going in or entering; the right to enter; a means or place of entering.
- What is an ingress? 
  In k8s, ingress is a way to exposure services of applications runnings inside a cluster to external clients. The Ingress function consists by three components:
  - The **Ingress API object**
    - Define and configure an ingress.
  - The **L7 load balancer or reverse proxy**
    - Routes traffic to the backend services.
  - The **Ingress controller**
    - Monitors the k8s API for Ingress objects and deloys and configures the load balancer or reverse proxy

> NOTE: L4 and L7 refer to layer 4 (Transport Layer; TCP, UDP) and layer 7 (Application Layer; HTTP) of the Open System Interconnection Model (OSI Model).

### 1.1 Introducing the Ingress object kind
To expose multiple services, we can use an ingresss object in k8s, in which we can config the L7 load balancer (proxy). Through the L7 proxies, we can make the services accessible to the outside world using the same entrypoint (ingress).

The following figure is an example of exposing multiple services using an ingress object.
![An ingress forwards external traffic to multiple services](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/12image001.png)

The ingress object will route incoming traffic to the backend services based on the HTTP request (HTTP Path: L7).

### 1.2 Introducing the Ingress controller and the reverse proxy
In K8s, we can choose different ingress controller as we want. Potential choices can be found at [Kubernetes Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). Some exapmes are: Nginx, Contour. Most of these ingress controllers use Nginx and Envoy as the reverse proxy.

- 






## Questions
- For the three components of ingress function, is the L7 load balancer in k8s cluster?
- Does ingress controller have load balancer?
- What are the differences between L4 and L7 load balancers?
- What are the differences between ingress and load balancer?
  - In AKS, if you run `kubectl describe svc kiada`, you will see that the external ip is named `LoadBalancer Ingress`. 
  - **This answer is provided by copilot and verification is needed.**  `This is the external ip of the load balancer. The load balancer is a L4 load balancer. It is not an ingress. The ingress is a L7 load balancer. It is a reverse proxy. It is a L7 load balancer because it can route traffic to the backend services based on the HTTP request. The load balancer is a L4 load balancer because it can only route traffic to the backend services based on the TCP/UDP port number.
- When we talk about ingress, there are three parts:
  - The ingress API object (ingress gateway pod in istio-system namespace)
  - The L7 load balancer or reverse proxy (proxy-v2 == envoy proxy)
  - The ingress controller (**What is the ingress controller in Istio?**)
- If the proxy doesn't send the request directly to a pod IP instead of the service IP for the pod, how does the proxy know which pod to send the request to? How does the proxy achieve the load balancing? 
  - Based on the service and endpoints objects, the proxy can know which pod to send the request to. The proxy can achieve the load balancing by using the iptables rules? 
  - [ ] **TODO**: Check the iptables rules in a node for a specific service.


## ToDos
- [ ] Reading notes on Azure Traffic Manager, Azure Standard Load Balancer, and Envoy Proxy. (L3, L4, L7 load balancers).
- [ ] Create a cluster with the Istio sample applications. Check out the ingress and ingress controller in the cluster.
- [ ] The way to create ingress controller in AKS is different from Chapter12. Following [this](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli) to create one in your cluster.
