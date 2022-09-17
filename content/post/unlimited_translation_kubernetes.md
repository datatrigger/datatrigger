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

In this post, we'll deploy our 3-container translation app on a Kubernetes cluster so it is publicly available at [translation.datatrigger.org](translation.datatrigger.org).

I am using a [managed cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview?authuser=3) on [Google Cloud Platform](https://cloud.google.com/)'s [GKE](https://cloud.google.com/kubernetes-engine).

Assuming you have a Kubernetes cluster up and running, deploying an app basically amount to writing a .yaml file called a manifest. It describes the collection of K8s object you want the cluster to deploy and maintain. [Here](https://github.com/datatrigger/unlimited-translation_kubernetes/blob/main/unlimited-translation-k8s.yaml) is the content of the translation app's manifest.

# Deployments

```Deployment``` objects. 

# Networks and ports

# Stateful microservices

# Volumes

# Resources

# Secrets

# Image pull policies

# Publishing on the Internet


