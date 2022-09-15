---
title: "A multi container ML app (3/3): Kubernetes"
summary: "Deploying the 3-container translation app on a Kubernetes cluster to get scalability and resilience."
date: 2022-09-15
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
1) [Deployment](#deployment)
2) [Networking](#networking)
3) [Docker secrets](#docker-secrets)
4) [Data Persistence](#data-persistence)
5) [Database dump](#database-dump)
6) [Startup order](#startup-order)
7) [Conclusion](#conclusion)