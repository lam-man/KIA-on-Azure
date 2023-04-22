# Managing Pods with Deployments

## 1. Introducing Deployments

Workloads are usually deployed through a Deployment. However, deployment is not the one to manage the pods. It is the underneath ReplicaSet that manages the pods. The deployment is used to manage the ReplicaSet.

![Relationship between Deployment, ReplicaSet and Pods](https://drek4537l1klr.cloudfront.net/luksa3/v-15/Figures/14_img_0001.png)