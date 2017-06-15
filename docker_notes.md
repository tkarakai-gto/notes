Largely based on Udemy course "Docker Mastery: The Complete Toolset From a Docker Captain" by [Bret Fisher](https://gemalto.udemy.com/user/bretfisher/).

# Docker Notes

Docker version format changed early 2017. Versions are now YY.MM  based (like Ubuntu) and you can choose *Stable* (slower) or *Edge* (faster) releases. This means that newer features drop faster in Edge releases, and fixes are backported for a longer timeline to Stable releases. Everyone wins.

* `docker <command> <subcommand> <options>`
* `docker` or `docker help` - Lists docker commands.
* `docker version`

*Client* - The command line client talking to the server using an API

*Server* - Docker Engine a.k.a. the Docker Server answering to the API requests and running docker functions

* `docker info` - More detailed info, including stats, drivers, etc.
* [`docs.docker.com`](https://docs.docker.com)


## Basics

*Images* - File(s) containing the application we want to run.

*Container* - An instance of the image running as a process.

You can have many containers running of the same image.

* `docker container run --publish 80:80 nginx`
Pulls latest image (is not in local image cache), starts a container based on the image, opens container port 80 forwarding to server port 80 and runs the default command.
* `docker container ls` - Lists running containers
* `docker container ls -a` - Lists all containers
* `docker image ls` - Lists all images
* `docker container run --publish 80:80 --name web-server --detach nginx` - Pulls image (is not in local cache), starts a container based on the image using the specified name, opens container port 80 forwarding to server port 80 and runs the default command, in the background (`--detach` or `-d`)
* `docker container logs nginx -f` - Tail of container stdout with "follow".
* `docker container stop nginx` - Stops running container
* `docker container start nginx` - Starts container
* `docker container rm -f nginx` - Removes container (with "force")


#### Docker Containers are *NOT* lightweight VMs

* They are just processes running on the host OS.
* Limited to what resources they can access.
* Exit when the process stops.

Compare PIDs reported by `docker top nginx` with PIDs reported by the regular Linux `ps aux | grep nginx`. They both show the same!

* `docker-machine ssh` - Teleport into the console of the Docker Engine (vs. running commands on the Client)

* https://github.com/mikegcoleman/docker101/blob/master/Docker_eBook_Jan_2017.pdf


#### Getting inside of an existing/running container

For stopped containers:

* `docker container start -ai ubuntu` - Starts a container with an interactive session (Ubuntu and alike)

For running containers:

* `docker container exec -it redis bash` - Starts interactive session (new process) of a running container (Ubuntu and other major distros)
* `docker container exec -it redis sh` - Starts interactive session (new process) of a running container (Alpine Linux)

* https://www.digitalocean.com/community/tutorials/package-management-basics-apt-yum-dnf-pkg


#### Basic Monitoring

* `docker container top nginx` - Lists processes of the given running container.
* `docker container stats` - Similar to Linux `top`, for all containers.
* `docker container stats nginx` - Similar to Linux `top`, for the specified container.
* `docker container inspect nginx` - Container metadata on container, including attached networks, MAC address, etc.


#### Quick temporary containers

`--rm` flag is so we don't need to clean up

* `docker container run --rm -d --name redis -p 6379:6379 redis:alpine` - Quick temporary Redis:alpine instance
* `docker container run --rm -d --name mysql --publish 3306:3306 -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql` - Quick temporary MySQL. Password is in the logs (`docker container logs mysql`).
* `docker container run --rm -it ubuntu` - Runs container in interactive mode and when done, it deleted the container.


## Networking Basics

By default containers use the "bridge" virtual network.

Containers on the same virtual network are free to talk to each other.

Best practice is to create a new virtual network for each app if they are not related

A container can be attached to multiple virtual networks (much like PCs can have multiple NICs). Or none.

A container can skip NAT and use the host networking `--net=host` (not recommended).

Then there are "network drivers" too...

* `docker container port nginx` - Shows port configuration of the container
* `docker container inspect --format '{{ .NetworkSettings.IPAddress }}' nginx` - Extracts the IP address of the container on the virtual network
* https://docs.docker.com/engine/admin/formatting/
* `docker network --help`
* `docker network ls` - Lists virtual networks
* `docker network create my_network [--driver ...]` - Create new virtual network (optionally with a driver)
* https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/
* `docker network inspect my_network` - JSON metadata about the virtual network, for example Subnet, default gateway, etc. Lists containers on it and their MAC addresses too.
* `docker container run --rm -d --network my_network nginx` - Run a container on a specific network (`--net` is a shorthand for `--network`)
* `docker network connect --help`
* `docker network connect my_network redis` - Connects container to the network (insert virtual NIC plugged into the given network)
* `docker network disconnect my_network redis` -Disconnects container from the network (remove virtual NIC plugged into the given network)
* `docker network rm my_network` - Delete virtual network


## Name Resolution

DNS is needed to reliably identify containers that are coming and going.

Container names are used as DNS names for host resolution on the same virtual network.

* `docker container exec -it nginx ping nginx2`

The default "bridge" network does NOT have the DNS server built into it. For that you will need use the `--link` (add a link to another container). Easier to create a new network :).

Using `docker compose` makes this mush easier by automatically creating virtual networks.

To create simple *round robin DNS name resolution* (a poor man's load balancer) use the `--net-alias` option. Container names must be unique, but aliases do not. Aliases will respond to DNS queries in a round-robin fashion.

* `docker container run --rm -d --net my_network --net-alias search elasticsearch:2` - (running the first container)
* `docker container run --rm -d --net my_network --net-alias search elasticsearch:2` - (running a second container, exact same command!)
* `docker container ls` - Notice the ports they use (9200, 9300), we are NOT exposing the ports on the host!
* `docker container run --rm --net my_network alpine nslookup search` - Should list both container IP addresses.
* `docker container run --rm --net my_network centos curl -s search:9200` - Calling elasticsearch by the alias. Repeatedly calling should alternate between the two containers. Check the server's `name` attribute to verify. *THIS DID NOT WORK FOR ME. IT ALWAYS CALLED THE SAME CONTAINER. ONLY SWITCHED TO THE OTHER ONE WHEN THE FIRST CONTAINER WAS STOPPED.*


## Building Images

*Image* is the application binaries and dependencies plus the metadata on how to run it. *Officially:* "An image is an ordered collection of root filesystem changes and the corresponding execution parameters for use within a container runtime".

* https://github.com/moby/moby/blob/master/image/spec/v1.md

There are no OS inside an image. No kernel, no kernel modules (e.g. drivers).

Images use the union filesystem which are layers on top of layers of filesystem changes.

* `docker history nginx` - Shows layers of changes (commands)

When starting a container Docker opens a read-write layer on top of the image. When a file is changed, that file is copied into the new layer. This is called  "copy-on-write" or COW.

* `docker inspect nginx` - This is the metadata about the image.

#### Tags

* `docker image tag --help`
* `docker image ls` - Look at the repository and tag columns

Tag is just a label for a specific image on docker hub. There can be multiple tag for the same image.

* `docker image tag nginx tkarakai/nginx:testing` - Tagging an existing image with another tag "testing". If no tag specified after the repository name, it defaults it to "latest".

"latest" does not necessarily mean "latest", it's just another label.

#### Docker Hub

* https://github.com/docker-library/official-images/tree/master/library
* `docker login tkarakai` - Log into docker hub by default. Stores auth key in `.docker/config.json` in Windows.
* `docker image push tkarakai/nginx:testing` - Push to docker hub.
* `docker image rm tkarakai/nginx:testing` - Remove tag (it figured based on the name that it's a tag, not an actual image) from the local repository.
* `docker logout`

Docker Hun supports private repositories. For that create the repo in docker hub first on the web interface with visibility "private", then push images.

#### Dockerfile Example 1

The `Dockerfile` is a recipe for creating a Docker image.

* `FROM` - Base image
* `ENV` - Set environment variables. This is the way set keys and values to inject.
* `RUN` - Execute command(s)
  * Use `&&` to concatenate commands to preserve FS layers.
  * Can run shell scripts, copy files, etc.

The proper way to log is *NOT* to log to a log file, and there is no syslogd aor other syslog server either, Docker actually handles all the logging for us. All we have to do is all our logs are spit out into STDIN and STDERR. See second `RUN` command below.

* `EXPOSE` - Expose ports withing the virtual network (not yet to the outside!)
* `CMD` - Command to run every time the container is started or restarted.

*Dockerfile*
```python
# NOTE: this example is taken from the default Dockerfile for the official nginx Docker Hub Repo
# https://hub.docker.com/_/nginx/
FROM debian:stretch-slim
# all images must have a FROM
# usually from a minimal Linux distribution like debain or (even better) alpine
# if you truly want to start with an empty container, use FROM scratch

ENV NGINX_VERSION 1.13.0-1~stretch
ENV NJS_VERSION   1.13.0.0.1.10-1~stretch
# optional environment variable that's used in later lines and set as envvar when container is running

RUN apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y gnupg1 \
	&& \
	NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
	found=''; \
	for server in \
		ha.pool.sks-keyservers.net \
		hkp://keyserver.ubuntu.com:80 \
		hkp://p80.pool.sks-keyservers.net:80 \
		pgp.mit.edu \
	; do \
		echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
		apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
	done; \
	test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
	apt-get remove --purge -y gnupg1 && apt-get -y --purge autoremove && rm -rf /var/lib/apt/lists/* \
	&& echo "deb http://nginx.org/packages/mainline/debian/ stretch nginx" >> /etc/apt/sources.list \
	&& apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y \
						nginx=${NGINX_VERSION} \
						nginx-module-xslt=${NGINX_VERSION} \
						nginx-module-geoip=${NGINX_VERSION} \
						nginx-module-image-filter=${NGINX_VERSION} \
						nginx-module-njs=${NJS_VERSION} \
						gettext-base \
	&& rm -rf /var/lib/apt/lists/*
# optional commands to run at shell inside container at build time
# this one adds package repo for nginx from nginx.org and installs it

RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log
# forward request and error logs to docker log collector

EXPOSE 80 443 8080
# expose these ports on the docker virtual network
# you still need to use -p or -P to open/forward these ports on host

CMD ["nginx", "-g", "daemon off;"]
# required: run this command when container is launched
# only one CMD allowed, so if there are mulitple, last one wins
```
#### docker image build

* `docker image build -t custom-nginx .` - Builds image (as "latest") in the current directory
* `docker image ls` - The new image should show up here.

*Exercise:*
1. Add a new exposed port to Dockerfile
1. Rebuild
1. Observe that it only rebuilds the last step (layer)
1. We have a new image id (overwriting the old one)
1. If you undo the `Dockerfile` changes and rebuild again, *ALL* steps are use from the cache!

The order of the instructions in `Dockerfile` is significant: `docker image build` will rebuilds every step after a change is detected in `Dockerfile`. Put most stable commands on the top, and most frequently changing commands towards the end.

#### Dockerfile Example 2

* `WORKDIR` - Staring root dir. Preferred to using `RUN cd /path`
* `COPY` - Copies file from the builder current directory to the image current dir.

*Dockerfile*
```shell
# this same shows how we can extend/change an existing official image from Docker Hub

FROM nginx:latest
# highly recommend you always pin versions for anything beyond dev/learn

WORKDIR /usr/share/nginx/html
# change working directory to root of nginx webhost
# using WORKDIR is prefered to using 'RUN cd /some/path'

COPY index.html index.html

# I don't have to specify EXPOSE or CMD because they're in my FROM
```

Inherits all instructions from the `nginx` `Dockerfile`.

* `docker image build -t nginx-with-html .`
* `docker container run --rm -d -p 80:80 nginx-with-html`
* Open `http://locahost` (or http://192.168.99.100 if using Docker Toolbox) to see the custom html.
* `docker image tag nginx-with-html tkarakai/nginx-with-html:latest` - Tag for fun, ready to be pushed to docker hub.

#### Dockerfile Example 3

https://github.com/tkarakai-gto/udemy-docker-mastery/blob/master/dockerfile-assignment-1
```java
FROM node:6-alpine

EXPOSE 3000

RUN apk add --update tini \
  && mkdir -p /usr/src/app

WORKDIR /usr/src/app

COPY package.json package.json

RUN npm install \
  && npm cache clean

COPY . .

CMD ["tini", "--", "node", "./bin/www"]
```

## Container Lifetime & Persistent Data

* [The 12-Factor App (Everyone Should Read: Key to Cloud Native App Design, Deployment, and Operation](https://12factor.net/)
* [12 Fractured Apps (A follow-up to 12-Factor, a great article on how to do 12F correctly in containers)](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c)
* [Intro to Immutable Infrastructure Concepts](https://www.oreilly.com/ideas/an-introduction-to-immutable-infrastructure)

*Containers are meant to be immutable and ephemeral (disposable) as a design goal. We do not change things once they are running, instead we re-deploy a whole new container.*

The container should not contain any application data mixed in with the application binaries.

#### Persistent Data Volumes

"Volume" is a container independent filesystem. When containers are deleted, volumes are left alone. Volumes need to be deleted independently.

In the `Dockerfile`:

* `VOLUME <path>` - Creates a new volume and assigns it to the path inside the container.

* `docker image inspect mysql` - Look for "Volumes". MySQL is an container using volumes, obviously :) .
* `docker container run --rm -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql` - Quick temporary MySQL.
* `docker container inspect mysql` - Look for "Volumes" and "Mounts" from where the the Docker host serves the volume to the container.
* `docker volume ls` - List with the unique IDs, not really useful
* `docker volume inspect 23b472102724d408a8374babf096d251a32fab5446db942d0f8dc0757e0c3b22` - JSON metadata of the volume (this is on the Docker Host!)

*Named Volumes*

* `docker container run --rm -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=true -v mysl-vol:/var/lib/mysql mysql` - Using "mysql-vol" as the name for the  `/var/lib/mysql` container dir. The container volume location can be found out from the Dockerfile (`VOLUME` line).

* `docker volume create--help` - This is required to be able to specify custom drivers and labels.

#### Bind Mounts

It is a mapping of the host file or directory to a container file or directory.

Skips the UFS, host files do overwrite files in container. Files are not deleted when the container is deleted.

Bind Mounts are host specific, they need specific data to be on the host to work, they cannot be used in `Dockerfile`, must be at `container run`.

Looks just like the volume option, but with an extra host full path specified and separated with a colon. Bind Mount starts with a `/`, Volume does not.

* `... run -v /Users/me/stuff:/path/in/contaier` (Mac/Linux)
* `... run -v //c/Users/me/stuff:/path/in/contaier` (Windows)

Useful for development when editing files on the host is more convenient that should be automatically picked up by the container.

* `docker container run --rm -d --name nginx -p 80:80 -v $(pwd):/usr/share/nginx/html nginx` - Use `$(pwd)` to substitute the current host dir.
* `docker container exec -it nginx bash` - Jump into the container
* `cd /usr/share/nginx/html` - Should see the files from the host

#### Database minor upgrade (exercise)

Instead of running the OS package management `update` of the db version, use named volumes. The new version should be the new version of the image, pointing to the original volume.

* Find volume path in the Dockerfile of postgres: https://hub.docker.com/r/library/postgres/tags/9.6.1/. It is `/var/lib/postgresql/data`
* `docker container run --rm -d --name psql -v psql:/var/lib/postgresql/data postgres:9.6.1`
* `docker container logs psql` - It should have the regular startup logs with the initial entries
* `docker container stop psql` - Stop old db version
* `docker container run --rm -d --name psql2 -v psql:/var/lib/postgresql/data postgres:9.6.2` - start new db version pointing to the existing volume
* `docker container logs psql2` - It should have a very short log, skipping the initialization

#### Jekyll exercise

`jekyll` is a static web site generator. [Jekyll, a Static Site Generator (just as background info, no need to install)](https://jekyllrb.com/) It is the tool generating github pages.

The point is developers only need to run the container (no need to install Ruby and dependencies), the developer can simply use Bind Mounts to point to his workstation's directory and edit the content files there

https://github.com/tkarakai-gto/udemy-docker-mastery/tree/master/bindmount-sample-1

* `docker container run --rm -p 80:4000 -v $(pwd):/site bretfisher/jekyll-serve` - run from the sample directory, starts the site generator
* Go to http://192.168.99.100/
* Update the post markdown file in the `_posts` directory
* Watch the server noticing the change and regenerating the web site
* Refresh the browser


## Compose

A way to configure relationships between containers, save `docker container run` settings in easy-to-read file, start/stop the entire environment with a single command.

It has two parts:

1. `YAML` formatted describing:
  * containers
  * networks
  * volumes
1. `docker-compose` CLI tool for dev/test automation (uses the YAML file)

* [The YAML Format: Sample Generic YAML File](http://www.yaml.org/start.html)
* [The YAML Format: Quick Reference](http://www.yaml.org/refcard.html)
* [Compose File Version Differences (Docker Docs)](https://docs.docker.com/compose/compose-file/compose-versioning/)
* [Docker Compose Release Downloads (good for Linux users that need to download manually)](https://github.com/docker/compose/releases)

#### docker-compose.yml

Versions are different, be careful.

This is just the default name, you can use another YAML file name with the `-f` option

* `docker-compose --help`

*Template YAML content*
```yaml
version: '3.1'  # if no version is specificed then v1 is assumed. Recommend v2 minimum

services:  # containers. same as docker run
  servicename: # a friendly name. this is also DNS name inside network
    image: # Optional if you use build:
    command: # Optional, replace the default CMD specified by the image
    environment: # Optional, same as -e in docker run
    volumes: # Optional, same as -v in docker run
  servicename2:

volumes: # Optional, same as docker volume create

networks: # Optional, same as docker network create
```

*Same as the above jekyll run command*

```yaml
version: '2'

# same as
# docker run -p 80:4000 -v $(pwd):/site bretfisher/jekyll-serve

services:
  jekyll:
    image: bretfisher/jekyll-serve
    volumes:
      - .:/site
    ports:
      - '80:4000'
```

*Wordpress example*
```yaml
version: '2'

services:

  wordpress:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_PASSWORD: example
      key: value
    volumes:
      - ./wordpress-data:/var/www/html

  mysql:
    image: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: example
    volumes:
      - ./mysql-data:/var/lib/mysql
```

*Example of a 3 node load balanced database cluster behind a Ghost web server*
```yaml
version: '3'

services:
  ghost:
    image: ghost
    ports:
      - "80:2368"
    environment:
      - URL=http://localhost
      - NODE_ENV=production
      - MYSQL_HOST=mysql-primary
      - MYSQL_PASSWORD=mypass
      - MYSQL_DATABASE=ghost
    volumes:
      - ./config.js:/var/lib/ghost/config.js
    depends_on:
      - mysql-primary
      - mysql-secondary
  proxysql:
    image: percona/proxysql
    environment:
      - CLUSTER_NAME=mycluster
      - CLUSTER_JOIN=mysql-primary,mysql-secondary
      - MYSQL_ROOT_PASSWORD=mypass

      - MYSQL_PROXY_USER=proxyuser
      - MYSQL_PROXY_PASSWORD=s3cret
  mysql-primary:
    image: percona/percona-xtradb-cluster:5.7
    environment:
      - CLUSTER_NAME=mycluster
      - MYSQL_ROOT_PASSWORD=mypass
      - MYSQL_DATABASE=ghost
      - MYSQL_PROXY_USER=proxyuser
      - MYSQL_PROXY_PASSWORD=s3cret
  mysql-secondary:
    image: percona/percona-xtradb-cluster:5.7
    environment:
      - CLUSTER_NAME=mycluster
      - MYSQL_ROOT_PASSWORD=mypass

      - CLUSTER_JOIN=mysql-primary
      - MYSQL_PROXY_USER=proxyuser
      - MYSQL_PROXY_PASSWORD=s3cret
    depends_on:
      - mysql-primary
```
