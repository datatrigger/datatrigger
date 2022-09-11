---
title: "A multi container Machine Learning application deployed on a Kubernetes cluster - Part 1/2: Docker"
summary: "Building a translation app by putting together 3 containerized microservices: a Flask frontend, a FastAPI backend and a MySQL database. Let's see how to properly dockerize each part and how we can connect them. Also covered: security with Docker secrets, CI/CD with GitHub Actions, data persistence with Docker volumes."
date: 2022-09-11
tags: ["docker", "container", "api", "nlp", "database", "flask", "fastapi", "python", "mysql", "secrets", "ci/cd", "registry"]
draft: true
---

Use the app at [translation.datatrigger.org](translation.datatrigger.org)

*Source code*:
* *[Docker Compose deployment](https://github.com/datatrigger/unlimited-translation_docker_swarm)*
* *[Flask frontend](https://github.com/datatrigger/unlimited_translation-frontend-swarm)*
* *[FastAPI backend](https://github.com/datatrigger/unlimited_translation-backend)*

### Introduction

As a non-German speaker living in Switzerland, I often need to quickly translate large texts, but I get annoyed by character limits on Google Translate or DeepL. Learning German may have been a *way* better call, but instead I decided to deploy a translation application. It's made of 3 containerized microservices:

* A Flask frontend to get inputs and display translations
* A FastAPI API backend to translate English text, using open-source models (SpaCy, Hugging Face)
* A MySQL database to store previous translations

In this post, we build and deploy the app on a single node with Docker. In part 2/2, the app will be deployed on a Kubernetes cluster.

### The project