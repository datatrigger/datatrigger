---
title: "A multi container ML app (2/3): deploying with Docker Compose"
summary: "Now that we have Docker containers, let's deploy them together with Docker Compose. Also covered: security with Docker secrets, data persistence with Docker volumes and dependency ordering."
date: 2022-09-15
tags: ["docker", "container", "compose", "secrets", "volume", "swarm"]
draft: false
---

*App:*

* [translate.vlgdata.io](https://translate.vlgdata.io)

*Source code:*
* [Flask frontend container](https://github.com/datatrigger/unlimited_translation-frontend-swarm)
* [FastAPI backend container](https://github.com/datatrigger/unlimited_translation-backend)
* [Docker Compose deployment](https://github.com/datatrigger/unlimited-translation_docker_swarm)
* [Kubernetes deployment](https://github.com/datatrigger/unlimited-translation_kubernetes)

*Content of this post:*
1) [Deployment](#1-deployment)
2) [Networking](#2-networking)
3) [Docker secrets](#3-docker-secrets)
4) [Data Persistence](#4-data-persistence)
5) [Database dump](#5-database-dump)
6) [Startup order](#6-startup-order)
7) [Conclusion](#7-conclusion)

### 1) Deployment

Let's see how we can deploy the translation app from [last post](https://blog.vlgdata.io/post/unlimited_translation_docker/) using Docker Compose. We need a folder with a ```docker-compose.yaml``` file at its root, see [this repository](https://github.com/datatrigger/unlimited-translation_docker_swarm) regarding the translation app's deployment.

The ```docker-compose.yaml``` file describes each microservices: which container image should be pulled, how to connect them together, should they be accessible on the network and through which port, etc...

```yaml
version: "3.9"
services:

  database:
    image: mysql:latest
    environment:
       MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
       MYSQL_DATABASE: translation
       MYSQL_USER_FILE: /run/secrets/db_user
       MYSQL_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_user
      - db_password
    volumes:
      - ./mysql-dump:/docker-entrypoint-initdb.d
      - unlimited_translation_database_volume:/var/lib/mysql

  backend_fastapi:
    image: datatrigger/unlimited-translation_backend_fastapi

  frontend_flask:
    image: datatrigger/unlimited-translation_frontend_flask:docker_swarm
    depends_on:
      - backend_fastapi
      - database
    ports:
      - "5000:80"
    environment:
      FLASK_DB_USER_FILE: /run/secrets/db_user
      FLASK_DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_user
      - db_password

secrets:
    db_root_password:
     external: true
    db_user:
     external: true
    db_password:
     external: true
   
volumes:
  unlimited_translation_database_volume:
```

Let's look at each part in details.

### 2) Networking

For the translation app to work, the 3 microservices need to be able to talk to each other on a network:
* The Flask frontend sends German text through HTTP requests to the FastAPI backend, who sends back the translated text
* The Flask frontend sends queries to the MySQL database, and gets the results back

As you can see in ```docker-compose.yaml``` file, each container is defined by a name, e.g. ```database``` or ```backend_fastapi```:

```yaml
version: "3.9"
services:

  database:
    ...

  backend_fastapi:
    ...

  frontend_flask:
    ...
```

When deploying the app, Docker Compose creates a network that connects the containers. Then, each container can be reached through its name. For instance, here is how we send requests to the backend from inside the Flask frontend container:

```python
requests.post('http://backend_fastapi/translate', headers=headers, json=json_data).json()['text_en']
```

To connect to the MySQL database, we simply specify ```database``` as the hostname in the connection parameters.

As we'll see in the next post, things get way more complicated when operating on a Kubernetes cluster.

### 3) Docker secrets

We need to provide a root password when creating the MySQL database. Ideally, we should grant the Flask frontend its own user and password so it can insert and query previous translations.

*Docker secrets* is a secure and convenient tool to manipulate sensitive data. The syntax is as follows:

```bash
printf "<secret_value>" | docker secret create <secret_name> -
```

Secrets can then be passed to containers in the ```docker-compose.yaml``` deployment file, e.g.:

```yaml
version: "3.9"
services:

  database:
    image: mysql:latest
    environment:
       MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
    secrets:
      - db_root_password
```
Quoting the [Docker docs](https://docs.docker.com/engine/swarm/secrets/):

> the decrypted secret is mounted into the container in an in-memory filesystem. The location of the mount point within the container defaults to /run/secrets/<secret_name> in Linux containers

This means that the value of the secret is then available in a file, inside the container, at ```/run/secrets/<secret_name>```. Instead of hardcoding this path inside scripts, it is best practice to pass the path as an environment variable, as in the example just above. This way, if the name of the secret changes at some point, you won't have to edit any of the scripts using this secret, but just the ```docker-compose.yaml``` file.

### 4) Data persistence

When a container stops running, all the data is lost. This is obviously a problem with some types of containers, e.g. databases. You want to be able to restore the data after the container was shut down, on purpose or by accident. Fortunately, [Docker volumes](https://docs.docker.com/storage/volumes/) are there to persist data generated by and used by Docker containers. Let's see an example with the MySQL database of the translation app:

```yaml
version: "3.9"
services:

  database:
    image: mysql:latest
    ...
    volumes:
      - unlimited_translation_database_volume:/var/lib/mysql   

volumes:
  unlimited_translation_database_volume:
```

At the bottom, the volume is created if it does not exist. Then, under the container definition section, we map this volume to the ```/var/lib/mysql``` folder inside the container, where MySQL stores the data. If we need to delete the volume at some point, we can run ```docker volume rm <volume_name>```.

### 5) Database dump

In the case of our translation app, we want to have a table ready to store our German texts and their translations up and running when the container is created. We need to be able to load the corresponding SQL instructions without manually connecting to the container each time it's launched. With the offical MySQL container, it is easy to load a [database dump](https://en.wikipedia.org/wiki/Database_dump) at startup.

We write a ```dump.sql``` script with the following instructions:

```sql
USE translation;
CREATE TABLE IF NOT EXISTS translations(id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, text_de LONGTEXT, text_en LONGTEXT);
```

After saving the script in the deployment repository, we can load it inside the MySQL container through the ```docker-compose.yaml``` file: map the folder containing the dump script to the folder ```/docker-entrypoint-initdb.d``` that lives inside the MySQL container. More details are available on the [MySQL official image repo](https://hub.docker.com/_/mysql).

*Loading ./mysql-dump/dump.sql instructions in the MYSQL container:*

```yaml
version: "3.9"
services:

  database:
    image: mysql:latest
    ...
    volumes:
      - ./mysql-dump:/docker-entrypoint-initdb.d
```

### 6) Startup order

In order for the app to be up and running as soon as the user can access the frontend, we need to ensure both the backend and the database are available first. Docker Compose allows to control startup and shutdown order with the ```depends_on``` option:

```yaml
frontend_flask:
    image: datatrigger/unlimited-translation_frontend_flask:docker_swarm
    depends_on:
      - backend_fastapi
      - database
```

### 7) Conclusion

That is all for the Docker Compose deployment. Technically, I ended up deploying with Docker Swarm on a single node, which is pretty much the same thing. The reason for that is the Docker secrets feature missing in Docker Compose, see details in the [project repo](https://github.com/datatrigger/unlimited-translation_docker_swarm).

In the [next post](https://blog.vlgdata.io/post/unlimited_translation_kubernetes/), we'll deploy the translation app on a Kubernetes cluster. This requires much more work but it makes an app scalable and resilient.