---
title: "A multi container ML app (1/2): Docker"
summary: "Building a translation app by putting together 3 containerized microservices: a Flask frontend, a FastAPI backend and a MySQL database. Let's see how to properly dockerize each part and how we can connect them. Also covered: security with Docker secrets, CI/CD with GitHub Actions, data persistence with Docker volumes."
date: 2022-09-11
tags: ["docker", "container", "api", "nlp", "database", "flask", "fastapi", "python", "mysql", "secrets", "ci/cd", "registry"]
draft: true
---

*App:*

* [translation.datatrigger.org](translation.datatrigger.org)

*Source code:*
* [Docker Compose deployment](https://github.com/datatrigger/unlimited-translation_docker_swarm)
* [Flask frontend container](https://github.com/datatrigger/unlimited_translation-frontend-swarm)
* [FastAPI backend container](https://github.com/datatrigger/unlimited_translation-backend)

### Introduction

As a non-German speaker living in Switzerland, I often need to quickly translate large texts, but I get annoyed by character limits on Google Translate or DeepL. Learning German may have been a *way* better call, but instead I decided to deploy a translation application. It's made of 3 containerized microservices:

* A Flask frontend to get inputs and display translations
* A FastAPI API backend to translate English text, using open-source models (SpaCy, Hugging Face)
* A MySQL database to store previous translations

In this post, we build and deploy the app on a single node with Docker. In part 2/2, the app will be deployed on a Kubernetes cluster.

### High-level overview of the app

1) On its main page, the Flask frontend takes German text as input and sends an HTTP request to the backend, which responds with the translated text. Then, both the original text and the translation are inserted inside the MySQL database.  
  
2) On its secondary page, the frontend takes a SQL query as input, fetch the results from the database and displays the result.

![microservices chart](/res/unlimited_translation_docker/unlimited_translation_chart.png)

### Before dockerizing

The first step is to build each microservice directly on our machine, without thinking about Docker. We use a virtual environment for each component (I like Python's standard ```venv```). This is much more convenient during the development phase since we do not have to rebuild a container every time we change something. In the meantime, we exactly know what modules each microservice needs.

```
cd path/to/microservice_folder
python3 -m venv .venv
source .venv/bin/activate
pip install ...
```

During this phase, I use the local network to connect the microservices together. I directly hardcode the URLs (like *http://localhost:80/*) in the components so they can talk to each other.

In this case, we have to write the Flask frontend and the FastAPI backend. For the MySQL database, we use the official container as is.

### Docker

Once the components

###### Dockerize each app

###