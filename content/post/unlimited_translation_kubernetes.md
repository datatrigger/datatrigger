---
title: "A multi container ML app (3/3): Kubernetes and Google Cloud Platform"
summary: "Deploying our 3-container translation app on a Kubernetes cluster to get scalability and resilience."
date: 2022-09-17
tags: ["kubernetes", "k8s", "gcp", "google cloud", "gke", "swarm"]
draft: false
---

*Source code:*
* [Flask frontend container](https://github.com/datatrigger/unlimited_translation-frontend-swarm)
* [FastAPI backend container](https://github.com/datatrigger/unlimited_translation-backend)
* [Kubernetes deployment](https://github.com/datatrigger/unlimited-translation_kubernetes)

*Content of this post:*
1) [Introduction](#1-introduction)
2) [Deployments](#2-deployments)
3) [Stateful microservices](#3-stateful-microservices)
4) [Resources](#4-resources)
5) [Secrets](#5-secrets)
6) [Image pull policies](#6-image-pull-policies)
7) [Networks and ports](#7-networks-and-ports)
8) [Publishing the app on the Internet](#8-publishing-the-app-on-the-internet)

# 1) Introduction

In this post, we'll deploy our 3-container translation app on a Kubernetes cluster and make it publicly available at [~~translate.vlg.engineer~~](https://translate.vlg.engineer) (The application is now down for cost reasons).

I am using a [managed cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview?authuser=3) on [Google Cloud Platform](https://cloud.google.com/)'s [GKE](https://cloud.google.com/kubernetes-engine).

Assuming you have a Kubernetes cluster up and running, deploying an app basically amount to writing a ```.yaml``` file called a manifest:

[*Manifest of the translation app*](https://github.com/datatrigger/unlimited-translation_kubernetes/blob/main/unlimited-translation-k8s.yaml)

It describes the collection of K8s object you want the cluster to manage at all time, regardless of failures or other disruptions.

# 2) Deployments

We use ```Deployment``` objects to deploy the frontend and backend of our translation app (for the database, we use a [```StatefulSet```](#stateful-microservices)), specifying:

* ```replicas```: number of replicated pods we want the k8s cluster to deploy
* ```selector```: describes which pods are targeted by the deployment
* ```template```: content of pods, e.g. container image, resources, environment variables...

Why is there a ```selector``` section when we already have a ```template``` describing the pods to be deployed? We might want to include already created pods in a new ```Deployment```. That is what the ```selector``` section allows to do.

We only scratch the surface with the translation app. ```Deployment``` objects can handle rollouts, i.e updates, using different strategies, e.g. *Recreate* (delete all containers v1, then deploy all containers v2) or *Rolling* (delete a container then replace with new version, one by one). The number of pods (```replicas```) can be scaled up and down depending on the demand. Autoscaling is available on cloud platforms (see [GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-autoscaler)).

# 3) Stateful microservices

### ```StatefulSet```

```Deployment```s are fine for stateless applications: if a frontend pod dies unexpectedly, it can just be replaced with a brand new container. On the other hand, this behavior is not acceptable with stateful microservices. A database is a textbook case of stateful app: no data loss is acceptable in case something happens to a pod/container. ```SatefulSet``` are designed to deploy stateful microservices, such as the MySQL database of the translation app.

With ```Deployment```s, pods are interchangeable, have random IPs and are all connected to the same ```PersistentVolume``` by design (if there is one). On the contrary, with ```StatefulSet```s each pod has a unique (ordinal) identifier. As a consequence, they can be reached individually and can be attached a unique ```PersistentVolume``` each. This allows consistent data replication in a master-slaves framework, where only the master has read/write access and the slaves can just read data.

Implementing master-slaves data replication is far from trivial: see [this example from the official Kubernetes documentation](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/). This is needed for data systems with strong reliability, availability and consistency requirements. For the translation app, we'll just deploy 1 ```replicas``` of a MySQL container. Even with ```replicas: 1``` pod, using a ```StatefulSet``` is needed as per the [GKE official documentation](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes):

> Even Deployments with one replica using ReadWriteOnce volume are not recommended. This is because the default Deployment strategy creates a second Pod before bringing down the first Pod on a recreate. The Deployment may fail in deadlock as the second Pod can't start because the ReadWriteOnce volume is already in use, and the first Pod won't be removed because the second Pod has not yet started. Instead, use a StatefulSet with ReadWriteOnce volumes.
>
> StatefulSets are the recommended method of deploying stateful applications that require a unique volume per replica. By using StatefulSets with PersistentVolumeClaim templates, you can have applications that can scale up automatically with unique PersistentVolumesClaims associated to each replica Pod.

To sum up: by using a ```StatefulSet```, data persistence and availability are guaranteed even in case of failure of the MySQL container (pod).

### Volumes

As discussed in the previous section, a volume is needed to persist the MySQL data in case of failure or even anticipated shutdown. ```StatefulSet``` makes it very convenient to allocate a unique ```PersistentVolume``` to each replicated pod. The desired volume is described in the ```volumeClaimTemplates``` section, then mounting is specified in the container ```template```:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  ...
spec:
  template:
    metadata:
      ...
    spec:
      containers:
      - name: mysql
        image: mysql:latest
        ...
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: data-volume
  volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 50Mi
  replicas: 1
  selector:
    ...
  serviceName: mysql-headless
```

We can also specify a ```StorageClassName``` in ```volumeClaimTemplates``` (see the [k8s documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) for an example). In the example above, no ```StorageClassName``` is given so the ```PersistentVolume```'s class will be the default ```StorageClass```. GKE's default ```StorageClass``` is *balanced persistent disk type (ext4)*, see [GKE's persitent volumes](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes). They are [backed by SSDs](https://cloud.google.com/compute/docs/disks).

```volumeClaimTemplates``` ensures each replicated pod gets its own volume with the specified characteristics. ```Deployment```s do not support ```volumeClaimTemplates``` since it would imply each pod can be identified (say a pod dies, which ```PersistentVolume``` should it connect to?).

With a ```Deployment```, a separate ```PersistentVolumeClaim``` is needed. If dynamic provisioning is not available, one must also describe a matching ```PersistentVolume``` in the manifest, and optionally a ```StorageClass``` definition. So, ```StatefulSet```s are more complex than ```Deployment```s but in the appropriate context (e.g. databases), they simplify the management of persistent data volumes.

### Database dump

In order for the MySQL database to be fully ready at pod creation, we want the right table to be defined at startup by running:

```sql
USE translation;
CREATE TABLE IF NOT EXISTS translations(id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, text_de LONGTEXT, text_en LONGTEXT);
```

This is a tiny database dump so using a volume would be an overkill here. We directly inject these 2 lines of SQL in the Kubernetes manifest with a ```ConfigMap``` object:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-initdb-configmap
data:
  initdb.sql: |
    USE translation;
    CREATE TABLE IF NOT EXISTS translations(id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, text_de LONGTEXT, text_en LONGTEXT);
```

Then, we mount this object to ```/docker-entrypoint-initdb.d``` as per the official [MySQL container documentation](https://hub.docker.com/_/mysql).

# 4) Resources

The [backend image](https://hub.docker.com/repository/docker/datatrigger/unlimited-translation_backend_fastapi) is very heavy (2 GiB) and a container instance requires significant amounts of cpu/memory to load the Machine Learning models (*Spacy* and *transformers* libraries) and compute translations. GKE's autopilot cluster does not automatically provision resources (but nodes according to requested resources). The [default values](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests) (CPU: 0.5 vCPU, Memory: 2 GiB, Ephemeral storage: 1 GiB) work fine for the Flask frontend container and for the MySQL Database. But the backend pods keep crashing.

### CPU and RAM

Let's run an instance of the FastAPI backend locally so we can evaluate the resources needed:

```bash
docker run -d -p 8080:80 datatrigger/unlimited-translation_backend_fastapi:latest
```

We can see how much resource this container uses by running ```docker stats``` in a separate container. At rest, the container already needs about 1.5 GiB of RAM. Let's make it translate [an extract of Goethe's Faust](https://github.com/martinth/mobverdb/blob/master/faust.txt):

```bash
curl -X 'POST' 'http://127.0.0.1:8080/translate' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"text_de": "Long german text..."}'
```

Now the resource consumed looks like that:

![resources consumed by FastAPI backend container](/res/unlimited_translation_kubernetes/fastapi_backend_resources.png)

CPU usage jumps to 200+ % (not really sure what that means!). Since we run a [4-core Intel i5](https://www.intel.com/content/www/us/en/products/sku/80811/intel-core-i54690k-processor-6m-cache-up-to-3-90-ghz/specifications.html), it is now clear that the default value of 0.5 core is too low. To ensure pods have enough resources at container startup, we can specify requirements in a ```Deployment``` as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  replicas: 1
  selector:
    matchLabels:
      name: backend-fastapi-pod
      app: unlimited-translation-app
  template:
    metadata:
      ...
    spec:
      containers:
      - name: backend-fastapi
        image: datatrigger/unlimited-translation_backend_fastapi:latest
        ...
        resources:
          requests:
            cpu: 4000m
            ephemeral-storage: 100Mi #for scratch space, logging or caching (GKE Autopilot default value: 1Gi)
            memory: 2Gi #(GKE Autopilot default value: 2Gi)
```

Here we request 4 vCPU, that is an 8-fold increase of the default value. It is also possible to specify ```limits```, but [GKE Autopilot only considers ```requests```](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests).

The ideal setting would be to [run the translations on a GPU](https://cloud.google.com/kubernetes-engine/docs/how-to/gpus) but that's a story for antoher day.

### Ephemeral storage

An quick way to roughly determine how much ephemeral storage a container needs is to run it, then ssh into it and evaluate the size of local folders.

```bash
docker run -d <container_image>
docker exec -it <container_id> sh
ls -l --block-size=M
```

# 5) Secrets

Just as for the [deployment with Docker Compose](https://vlg.engineer/post/unlimited_translation_deploy_with_docker_compose/), we need to pass a root password to the MySQL container and a user/password to the Flask frontend microservice. First, we define the credentials in the CLI of the Kubernetes cluster:

```bash
kubectl create secret generic mysql-db-secret \
  --from-literal=MYSQL_ROOT_PASSWORD='<root_password>' \
  --from-literal=MYSQL_USER='<user>' \
  --from-literal=MYSQL_PASSWORD='<password>'
```

Then, we can pass the values to the pods as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  replicas: 1
  selector:
    ...
  template:
    spec:
      containers:
      - name: frontend-flask
        image: datatrigger/unlimited-translation_frontend_flask:k8s
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-db-secret
              key: MYSQL_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-db-secret
              key: MYSQL_PASSWORD
```

Finally, we retrieve these values inside the container like so:

```python
import os

db_user = os.environ.get('DB_USER')
db_password = os.environ.get('DB_PASSWORD')
db_name = os.environ.get('DB_NAME')
```

# 6) Image pull policies

As per the [Kubernetes documentation](https://kubernetes.io/docs/concepts/containers/images/):

> When you first create a Deployment, StatefulSet, Pod, or other object that includes a Pod template, then by default the pull policy of all containers in that pod will be set to IfNotPresent if it is not explicitly specified. This policy causes the kubelet to skip pulling an image if it already exists.

It is actually a bit more complicated, depending on the tag of the image being ```latest``` or not.

Since the tag of the Flask frontend container is not ```latest```, the image is not pulled if it exists on the cluster. So, when I was developing the translation app and regularly updating the frontend, I had to change th image pull policy to ```Always```, otherwise I would not see the changes. This is done by adding the line below to the pod's template:

```yaml
imagePullPolicy: Always
```

# 7) Networks and ports

The 3 microservices need to be able to talk to each other in the Kubernetes cluster:
* The Flask frontend sends German text through HTTP requests to the FastAPI backend, who sends back the translated text
* The Flask frontend sends queries to the MySQL database, and gets the results back

We also need an external connection from outside the cluster for users to connect to the frontend.

Network connection are not handled inside ```Deployment``` or ```StatefulSet```, but through independent ```Service``` objects.

### ClusterIP

This is the default type of ```Service```, allowing the targeted pods to be reached within the Kubernetes cluster. ```ClusterIP``` services also provide load-balancing to ensure the requests to the pods 'covered' by the service are equally distributed.

We use a ClusterIP for the FastAPI backend pods since only frontend pods connect to them:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-fastapi-service
  labels:
    name: backend-fastapi-service
    app: unlimited-translation-app
spec:
  #type: ClusterIP #not needed since 'ClusterIP' is the default value
  ports:
  - port: 80
    #targetPort: 80 #not needed here: same value as `port` by default
  selector:
    name: backend-fastapi-pod
    app: unlimited-translation-app
```

This way, we can send http requests from inside the Flask frontend containers as follows:

```python
json_data = {'text_de': text_de} #`text_de` is the user's input text in German
text_en = requests.post('http://backend-fastapi-service/translate', headers=headers, json=json_data).json()['text_en']
```

*Note: the default port for http requests is 80, which is also the port the ClusterIP Service listens to. That's why it is not specified here.*

### Headless Services

```ClusterIP``` is an appropriate type of ```Service``` for ```Deployment```s since it does not tell the difference between pods. But, with a replicated database deployed as a ```StatefulSet```, we would want to reach specific pods, e.g.:

* The master pod for writing data
* The slaves for reading data

That is what a headless service makes possible. It is defined by the property ```clusterIP: None```. Specific pods are then reached by their index (0, 1, 2...), e.g.:

```python
cnx = mysql.connector.connect(user=db_user, password=db_password, host='mysql-stateful-0.mysql-headless', database=db_name)
```

### NodePort/LoadBalancer

In order to be reached from outside the Kubernetes cluster, a ```Service``` must be of type ```NodePort``` or ```LoadBalancer``` (i.e ```NodePort``` with load-balancing). In such a case, 3 ports are to be specified:
* ```port```: The service's port, i.e the one to use in when reaching the service's pods
* ```targetPort```: the port exposed by the application inside the pods' container
* ```nodePort```: the port to reach the Kubernetes cluster from the outside

A ```port``` must be specified. If not provided, ```targetPort``` will be set to the same value as ```port``` and ```nodePort``` will be set a random value in the valid range.

I used a ```LoadBalancer``` during the development of the translation app (commented out in the [manifest](https://github.com/datatrigger/unlimited-translation_kubernetes/blob/main/unlimited-translation-k8s.yaml)). It conveniently gets you an IP to connect to your app. The IP is accessible by running the command ```kubectl get svc -o wide```.

# 8) Publishing the app on the Internet

In order to publish the app, we need yet another type of Kubernetes object: ```Ingress```.

> Ingress is a Kubernetes resource that encapsulates a collection of rules and configuration for routing external HTTP(S) traffic to internal services.

The high-level steps to make our app available from the domain name [translate.vlg.engineer](https://translate.vlg.engineer) are as follows:

0) Get the domain name

1) Reserve a static external IP address. I [reserved a Google IP addresses](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address#console) and named it ```translation-ip```

2) [Configure ```NodePort``` service and ```Ingress``` to connect the above IP to the app](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer) running on a Kubernetes cluster

3) [Implement a secure HTTPS connection with SSL certificates](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl). I use [Google-managed SSL certificates](https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs)

4) Redirect HTTP to HTTPS to force secure connections, see [this stackoverflow question](https://stackoverflow.com/questions/37001557/how-to-force-ssl-for-kubernetes-ingress-on-gke) and the manifest source code

5) [Point the domain name to the static external IP address](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) by setting appropriate DNS A records

~~The translation app is now deployed on Kubernetes at [translate.vlg.engineer](https://translate.vlg.engineer)!~~ - *Update*: The app has been taken down for cost reasons.

# The end

Thanks for reading!


