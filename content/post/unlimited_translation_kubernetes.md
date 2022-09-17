---
title: "A multi container ML app (3/3): Kubernetes"
summary: "Deploying our 3-container translation app on a Kubernetes cluster to get scalability and resilience."
date: 2022-09-17
tags: ["kubernetes", "k8s", "gcp", "google cloud", "gke", "swarm"]
draft: true
---

*App:*

* [translation.datatrigger.org](translation.datatrigger.org)

*Source code:*
* [Flask frontend container](https://github.com/datatrigger/unlimited_translation-frontend-swarm)
* [FastAPI backend container](https://github.com/datatrigger/unlimited_translation-backend)
* [Kubernetes deployment](https://github.com/datatrigger/unlimited-translation_kubernetes)

*Content of this post:*
1) [](#)
2) [](#)
3) [](#)

# Introduction

In this post, we'll deploy our 3-container translation app on a Kubernetes cluster and make it publicly available at [translation.datatrigger.org](translation.datatrigger.org).

I am using a [managed cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview?authuser=3) on [Google Cloud Platform](https://cloud.google.com/)'s [GKE](https://cloud.google.com/kubernetes-engine).

Assuming you have a Kubernetes cluster up and running, deploying an app basically amount to writing a .yaml file called a manifest:

[*Manifest of our translation app*](https://github.com/datatrigger/unlimited-translation_kubernetes/blob/main/unlimited-translation-k8s.yaml)

It describes the collection of K8s object you want the cluster to manage at all time, regardless of failures or other disruptions.

# Deployments

We use ```Deployment``` objects to deploy the frontend and backend of our translation app (for the database, we use a [```StatefulSet```](#stateful-microservices)), specifying:

* ```replicas```: number of replicated pods we want the k8s cluster to deploy
* ```selector```: describes which pods are targeted by the deployment
* ```template```: content of pods, e.g. container image, resources, environment variables...

Why is there a ```selector``` section when we already have a ```template``` describing the pods to be deployed? We might want to include already created pods in a new ```Deployment```. That is what the ```selector``` section allows to do.

We only scratch the surface with the translation app. ```Deployment``` objects can handle rollouts, i.e updates, using different strategies, e.g. *Recreate* (delete all containers v1, then deploy all containers v2) or *Rolling* (delete a container then replace with new version, one by one). The number of pods (```replicas```) can be scaled up and down depending on the demand. Autoscaling is available on cloud platforms (see [GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-autoscaler)).

# Stateful microservices

```Deployment```s are fine for stateless applications: if a frontend pod dies unexpectedly, it can just be replaced with a brand new container. On the other hand, this behavior is not acceptable with stateful microservices. A database is a textbook case of stateful app: no data loss is acceptable in case something happens to a pod/container. ```SatefulSet``` are designed to deploy stateful microservices, such as the MySQL database of the translation app.

With ```Deployment```s, pods are interchangeable, have random IPs and are all connected to the same ```PersistentVolume``` by design (if there is one). On the contrary, with ```StatefulSet```s each pod has a unique (ordinal) identifier. As a consequence, they can be reached individually and can be attached a unique ```PersistentVolume``` each. This allows consistent data replication in a master-slaves framework, where only the master has read/write access and the slaves can just read data.

Implementing master-slaves data replication is far from trivial: see [this example from the official Kubernetes documentation](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/). This is needed for data systems with strong reliability, availability and consistency requirements. For the translation app, we'll just deploy 1 ```replicas``` of a MySQL container. Even with ```replicas: 1``` pod, using a ```StatefulSet``` is needed as per the [GKE official documentation](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes):

> Even Deployments with one replica using ReadWriteOnce volume are not recommended. This is because the default Deployment strategy creates a second Pod before bringing down the first Pod on a recreate. The Deployment may fail in deadlock as the second Pod can't start because the ReadWriteOnce volume is already in use, and the first Pod won't be removed because the second Pod has not yet started. Instead, use a StatefulSet with ReadWriteOnce volumes.
>
> StatefulSets are the recommended method of deploying stateful applications that require a unique volume per replica. By using StatefulSets with PersistentVolumeClaim templates, you can have applications that can scale up automatically with unique PersistentVolumesClaims associated to each replica Pod.

To sum up: by using a ```StatefulSet```, data persistence and availability are guaranteed even in case of failure of the MySQL container (pod).

# Volumes



# Resources

# Secrets

# Image pull policies

# Networks and ports

# Publishing the app on the Internet


