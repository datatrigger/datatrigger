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

We only scratch the surface with the translation app. ```Deployment``` objects can handle rollouts, i.e updates, using different strategies, e.g.:
* Recreate: delete all containers v1, then deploy all containers v2
* Rolling: delete a container then replace with new version, one by one

# Networks and ports

# Stateful microservices

# Volumes

# Resources

# Secrets

# Image pull policies

# Publishing on the Internet


