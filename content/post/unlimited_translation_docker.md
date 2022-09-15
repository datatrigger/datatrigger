---
title: "A multi container ML app (1/3): Docker"
summary: "Building a translation app by putting together 3 containerized microservices: a Flask frontend, a FastAPI backend and a MySQL database. Let's skim through the development process and the containerization. Also covered: Docker registry and CI/CD with GitHub Actions."
date: 2022-09-11
tags: ["docker", "container", "api", "nlp", "database", "flask", "fastapi", "python", "sql", "ci/cd", "registry"]
draft: false
---

*App:*

* [translation.datatrigger.org](translation.datatrigger.org)

*Source code:*
* [Flask frontend container](https://github.com/datatrigger/unlimited_translation-frontend-swarm)
* [FastAPI backend container](https://github.com/datatrigger/unlimited_translation-backend)
* [Docker Compose deployment](https://github.com/datatrigger/unlimited-translation_docker_swarm)
* [Kubernetes deployment](https://github.com/datatrigger/unlimited-translation_kubernetes)

*Content of this post:*
1) [Introduction](#introduction)
2) [High-level overview of the app](#high-level-overview-of-the-app)
3) [Before dockerizing](#before-dockerizing)
4) [Docker](#docker)
5) [Docker registry](#docker-registry)
6) [CI/CD](#ci-cd-with-github-actions)

### Introduction

As a non-German speaker living in Switzerland, I often need to quickly translate large texts, but I get annoyed by character limits on Google Translate or DeepL. Learning German may have been a *way* better call, but instead I decided to deploy a translation application. It's made of 3 containerized microservices:

* A Flask frontend to get inputs and display translations
* A FastAPI API backend to translate English text, using open-source models (SpaCy, Hugging Face)
* A MySQL database to store previous translations

In this post, we build the containers. In [part 2/3](https://www.datatrigger.org/post/unlimited_translation_deploy_with_docker_compose/), we'll deploy the app on a single node with Docker Compose. Finally, the app will be deployed on a Kubernetes cluster in [part 3/3](https://www.datatrigger.org/post/unlimited_translation_kubernetes/).

### High-level overview of the app

1) On its main page, the Flask frontend takes German text as input and sends an HTTP request to the backend, which responds with the translated text. Then, both the original text and the translation are inserted inside the MySQL database.  
  
2) On its secondary page, the frontend takes a SQL query as input, fetch the results from the database and displays the result.

![microservices chart](/res/unlimited_translation_docker/unlimited_translation_chart.png)

### Before dockerizing

The first step is to build each microservice directly on our machine, without thinking about Docker. I use a virtual environment for each component (I like Python's standard ```venv```). This is much more convenient during the development phase since there is no need to rebuild a container every time something changes. In the meantime, I am in control of the modules/python version each microservice needs to work properly.

Below is the ```venv``` syntax to create a virtual environment named *.venv*:

```bash
cd path/to/microservice_folder
python3 -m venv .venv
source .venv/bin/activate
pip install ...
```  

During this phase, I use the local network to connect the microservices together. I directly hardcode the URLs (like *http://localhost:80/*) in the components so they can talk to each other.

In this case, we have to write the Flask frontend and the FastAPI backend. For the MySQL database, we'll use the official container as is.

### Docker

Once the app works on a local machine, we can start to build container images.

#### How to Dockerize a microservice

My method is as follows:

1) Get python's version: ```python --version``` or ```python3 --version```  

2) Write the list of **imported** packages in a ```requirements.txt``` file

I've seen people using ```pip freeze > requirements.txt``` but I prefer to avoid this because it lists every single module in your environment. This includes potentially unnecessary modules that you might have tried during development, or dependencies that do not need to be explicitly listed. So, I manually list the packages I actually **import** in my scripts: the ```requirements.txt``` file is much shorter and readable this way. There is actually a module called [pipreqs](https://github.com/bndr/pipreqs) to automate this process.

3) Create a folder/repository with the following structure:

```
/docker_image_repo
├── workdir
│   ├── script_1.py
│   ├── script_2.py
|   |── some_folder
|   |...
│   ├── requirements.txt
├── Dockerfile
```
#### The Dockerfile

With the above steps completed, it is now pretty straightforward to write the ```Dockerfile```. Check out the source for the [Flask frontend image](https://github.com/datatrigger/unlimited_translation-frontend-swarm) or the [FastAPI backend image](https://github.com/datatrigger/unlimited_translation-backend). Let's look at the backend's Dockerfile in detail:

```docker {linenos=table}  
FROM python:3.10 #Start from a Debian distirbution with the exact python version needed  
COPY /workdir /workdir #Copy the scripts and source code files  
WORKDIR workdir #Set the working directory as... The workdir folder  
RUN pip install --no-cache-dir --upgrade -r requirements.txt && python pull_nlp_models.py #Set the python environment + download NLP models  
CMD ["uvicorn", "backend_fastapi:app", "--host", "0.0.0.0", "--port", "80"] #Run the FastAPI microservice  
```  

#### Keep it light

In order to translate with no character limit, the FastAPI backend works as follows:
1) **Segment** the input text in sentences using a [SpaCy model](https://spacy.io/models/de#de_dep_news_trf)
2) **Translate** sentences one by one with [🤗 Hugging Face](https://huggingface.co/)'s ```transformers``` library

We decompose the work this way because transformers models cannot process long text: they usually take no more than 512 or 1024 words/tokens at a time.

We could put these models in the source code of the Docker image along with the .py scripts, but instead we pull them at image build: through the ```requirements.txt``` file for the Spacy model and through the ```pull_nlp_models.py``` script for the translation model. We proceed this way for several reasons:

* The models are very heavy and cannot fit in a standard GitHub repository (100 Mo max). Yet we need to store this code in a repo to implement CI/CD
* It is much easier to update or change the models this way

Overall, the point is to keep the source code of a Docker image as light and simple as possible.

### Docker registry

The point of containers is that they can run anywhere, be it someone's local machine, a server or a Kubernetes cluster. So they have to be accessible from anywhere. This is exactly the point of a container registry. The syntax is as follows:

```bash
cd <docker_image_repo>
docker build . -t <registry>/<image name>:<image tag>
docker push <image name>:<image tag>
```

For the translation project, the Docker images live on my personal [DockerHub account](https://hub.docker.com/u/datatrigger). When we deploy the app later on, the images will be pulled from there to run the containers.

### CI/CD with GitHub Actions

It would be nice not having to manually build the image and push it to the registry each time we change something in the source code. This is where GitHub Actions comes in: each push on the repository automatically triggers a build of the image and pushes it to the registry. The detailed steps to implement CI/CD are very well documented on [Docker's official docs](https://docs.docker.com/ci-cd/github-actions/).

It actually gets even better than that. If you look at our [FastAPI backend image](https://hub.docker.com/repository/docker/datatrigger/unlimited-translation_backend_fastapi) for instance:

![FastAPI Docker image](/res/unlimited_translation_docker/fastapi_backend_image.png)

You'll see the image repository has actually two different tags: *buildcache* and *latest*.

*latest* is the tag of the actual image of the container. What about *buildcache*? GitHub Actions looks at this file to know which parts of the image are impacted by the latest changes pushed to the source repository. This allows not to re-build the entire image at each push, but just the impacted layers. This saves a huge amount of time, especially with heavy images like the [FastAPI backend](https://hub.docker.com/repository/docker/datatrigger/unlimited-translation_backend_fastapi) (2+ Go) with heavy ML models embedded in it.

### Deployment

After developing and testing our microservices locally, we built container images and pushed them to a registry in a CI/CD framework. We can now easily maintain them and pull them from anywhere for deployment. Let's see how that goes in the [next post](https://www.datatrigger.org/post/unlimited_translation_deploy_with_docker_compose/).