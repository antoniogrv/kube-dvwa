# DVWA over Kubernetes

> An array of Kubernetes implementations (Kompose, StatefulSet, Chart) of DVWA.

[DVWA](https://github.com/digininja/DVWA?tab=readme-ov-file#damn-vulnerable-web-application) is a vulnerable-by-design web application, featuring CSRF attacks, SQL injections, command injections, stored/reflected XSS attacks and more. It is usually used for educational purposes, and consists of a PHP application as well as a MySQL/MariaDB database.

## References

The Docker distribution of DVWA is supplied using Docker Compose [[ref]](https://github.com/digininja/DVWA/blob/master/compose.yml).

<details>
  <summary>compose.yml</summary>

```yaml
volumes:
  dvwa:

networks:
  dvwa:

services:
  dvwa:
    build: .
    image: ghcr.io/digininja/dvwa:latest
    # Change `always` to `build` to build from local source
    pull_policy: always
    environment:
      - DB_SERVER=db
    depends_on:
      - db
    networks:
      - dvwa
    ports:
      - 127.0.0.1:4280:80
    restart: unless-stopped

  db:
    image: docker.io/library/mariadb:10
    environment:
      - MYSQL_ROOT_PASSWORD=dvwa
      - MYSQL_DATABASE=dvwa
      - MYSQL_USER=dvwa
      - MYSQL_PASSWORD=p@ssw0rd
    volumes:
      - dvwa:/var/lib/mysql
    networks:
      - dvwa
    restart: unless-stopped
```
</details>

The `ghcr.io/digininja/dvwa:latest` CRI-compatible image is built using a simple Dockerfile [[ref]](https://github.com/digininja/DVWA/blob/master/Dockerfile).
<details>
  <summary>Dockerfile</summary>

```yaml
FROM docker.io/library/php:8-apache

LABEL org.opencontainers.image.source=https://github.com/digininja/DVWA
LABEL org.opencontainers.image.description="DVWA pre-built image."
LABEL org.opencontainers.image.licenses="gpl-3.0"

WORKDIR /var/www/html

# https://www.php.net/manual/en/image.installation.php
RUN apt-get update \
 && export DEBIAN_FRONTEND=noninteractive \
 && apt-get install -y zlib1g-dev libpng-dev libjpeg-dev libfreetype6-dev iputils-ping \
 && apt-get clean -y && rm -rf /var/lib/apt/lists/* \
 && docker-php-ext-configure gd --with-jpeg --with-freetype \
 # Use pdo_sqlite instead of pdo_mysql if you want to use sqlite
 && docker-php-ext-install gd mysqli pdo pdo_mysql

COPY --chown=www-data:www-data . .
COPY --chown=www-data:www-data config/config.inc.php.dist config/config.inc.php
```
</details>

## Docker Installation

[As per the instructions by the official authors](https://github.com/digininja/DVWA/tree/master#docker), running DVWA on Docker is as simple as cloning the repository, executing `docker compose up -d` and accessing the application via `localhost:4280`. Default credentials to log in are `admin:admin`.

The [Apache webserver](https://hub.docker.com/layers/library/php/8-apache/images/sha256-20a5a87a4752077ff5dc3621a1c107295d6c976e09e95aa5f8fa369471922599?context=explore) runs on the host post 4280. Since the PHP service belongs to the same Docker Network as MySQL, it can reach the database by internally referencing the `db:3306` virtual hostname. No additional configuration is needed.


## Kubernetes Implementations

When deciding to distribute an application on Kubernete, we have multiple options to choose from.

### Kompose

Since the application is originally distributed using a Docker Compose configuration file, the easiest way to accomplish this task is to use [Kompose](https://kubernetes.io/docs/tasks/configure-pod-container/translate-compose-kubernetes/), a conversion tool for Docker Compose files to Kubernetes resources.

Before generating the Kubernetes resources, we must fine-tune DVWA's `compose.yml` to respect [Kompose's conversion matrix](https://kompose.io/conversion/). Specifically:
- Adding `version: 3` at the top of the configuration file, as it is required
- Removing or commenting `pull_policy: always` in `services.dvwa`, as it is not supported
- Changing `restart_policy: unless-stopped` to `restart_policy: always` in both `services.dvwa` and `services.db`, as it is equivalent
- Changing `127.0.0.1:4280:80` to `4280:80`, as it is not needed
- Adding `ports[3306:3306]` to `services.db`, as it is required to create a database-attached ClusterIP service 

Applying these fixes is mandatory. The correct `compose.yml` is the following:

<details>
  <summary>compose.yml</summary>
  
  ```yaml
  version: '3'

	volumes:
	  dvwa:

	networks:
	  dvwa:

	services:
	  dvwa:
	    build: .
	    image: ghcr.io/digininja/dvwa:latest
	    environment:
	      - DB_SERVER=db
	    depends_on:
	      - db
	    networks:
	      - dvwa
	    ports:
	      - 4280:80
	    restart: always

	  db:
	    image: docker.io/library/mariadb:10
	    environment:
	      - MYSQL_ROOT_PASSWORD=dvwa
	      - MYSQL_DATABASE=dvwa
	      - MYSQL_USER=dvwa
	      - MYSQL_PASSWORD=p@ssw0rd
	    networks:
	      - dvwa
	    volumes:
	      - dvwa:/var/lib/mysql
	    restart: always
      ports: 
        - 3306:3306
  ```
</details>

The Kubernetes resources can then be produced via `kompose convert -f . --with-kompose-annotation=false -v` and created on any cluster (e.g. a Minikube local cluster) using `kubectl create -f kompose`. Since the DVWA pod forwards connections from port 4280 to container port 80 (the Apache webserver), we can access the web application by port-forwarding a free local port on the host machine to port 80, by using the following command: `kubectl port-forward deployment/dvwa 4281:80`. 

The application will be finally accessible on `localhost:4281`.

```
git clone https://github.com/antoniogrv/kube-dvwa.git
kubectl create -f kompose
kubectl port-forward deployment/dvwa 4281:80
```

### StatefulSets

It is also possible to use a StatefulSet for MySQL and a distinct Deployment to distribute the PHP application.

The StatefulSet is connected to a ClusterIP Service in order to be resolved by the DVWA pods.

This solution is preferable to the latter as MySQL/MariaDB-backed Pods will retain the same PersistenceVolume(s) across reschedulings by the Control Plane.

Execute `kubectl create -f statefulset` to generate the Kubernetes resources, and use `kubectl port-forward deployment/dvwa 4281:80` to access the application on `localhost:4281`.

```
git clone https://github.com/antoniogrv/kube-dvwa.git
kubectl create -f statefulset
kubectl port-forward deployment/dvwa 4281:80
```

### Helm Chart

An even better solution is to wrap the StatefulSet implementation in a simple Helm chart, while also making it possible to configure most options for both the web application and the database via a straightforward `values.yaml` file. Please refer to [this repository](https://github.com/antoniogrv/kube-dvwa-helm-chart) for further instructions.

```
git clone https://github.com/antoniogrv/kube-dvwa-helm-chart.git
helm install kube-dvwa-helm-chart
kubectl port-forward deployment/dvwa 4281:80
```