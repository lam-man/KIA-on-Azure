# Exposing Services with Ingress

## Introducing Ingresses

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

## Questions
- For the three components of ingress function, is the L7 load balancer in k8s cluster?
- Does ingress controller have load balancer?
- What are the differences between L4 and L7 load balancers?

## ToDos