# Docker Notes

Docker version format changed early 2017. Versions are now YY.MM  based (like Ubuntu) and you can choose *Stable* (slower) or *Edge* (faster) releases. This means that newer features drop faster in Edge releases, and fixes are backported for a longer timeline to Stable releases. Everyone wins.

`docker <command> <subcommand> <options>`

`docker` or `docker help` - Lists docker commands.

`docker version`
*Client* - The command line client talking to the server using an API
*Server* - Docker Engine a.k.a. the Docker Server answering to the API requests and running docker functions

`docker info` - More detailed info, including stats, drivers, etc.

[`docs.docker.com`](https://docs.docker.com)


## Basics

*Images* - File(s) cntaining the application we want to run.
*Container* - An instance of the image running as a process.
You can have many containers running of the same image.

`docker container run --publish 80:80 nginx`
Pulls latest image (is not in local image cache), starts a container based on the image, opens container port 80 forwarding to server port 80 and runs the default command.

`docker container ls` - Lists running containers
`docker container ls -a` - Lists all containers

`docker image ls` - Lists all images

`docker container run --publish 80:80 --name webServer --detach nginx` - Pulls image (is not in local cache), starts a container based on the image using the specified name, opens container port 80 forwarding to server port 80 and runs the default command, in the background (`--detach` or `-d`)

`docker container logs nginx -f` - Tail of container stdout with "follow".

`docker container stop nginx` - Stops running container
`docker container start nginx` - Starts container

`docker container rm -f nginx` - Removes container (with "force")


## Docker Containers are *NOT* lightweight VMs

* They are just processes running on the host OS
* Limited to what resources they can access
* Exit when the process stops

Compare PIDs reported by `docker top nginx` with PIDs reported by the regular Linux `ps aux | grep nginx`. They both show the same!

`docker-machine ssh` - Teleport into the console of the Docker Engine (vs. running commands on the Client)


## Getting inside of an existing/running container

Not running containers:
`docker container start -ai ubuntu` - Starts a container with an interactive session (Ubuntu and alike)

Runnig containers:
`docker container exec -it redis bash` - Starts interactive session (new process) of a runnig container (Ubuntu and other major distros)
`docker container exec -it redis sh` - Starts interactive session (new process) of a runnig container (Alpine Linux)


## Basic Monitoring

`docker container top nginx` - Lists processes of the given running container.

`docker container stats` - Similar to Linux `top`, for all containers.
`docker container stats nginx` - Similar to Linux `top`, for the secified container.

`docker container inspect nginx` - Container metadata on container, including attached networks, MAC addres, etc.


## Quick temporary containers

`--rm` flag is so we don't need to clean up

`docker container run --rm -d --name redis -p 6379:6379 redis:alpine` - Quick temporary Redis:alpine instance
`docker container run --rm -d --name mysql --publish 3306:3306 -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql` - Quick temporary MySQL. Password is in the logs (`docker container logs mysql`).
`docker container run --rm -it ubuntu` - Runs container in intercative mode and when done, it deleted the container.


## Networking Basics

* By default containers use the "bridge" virtual network.
* Containers on the same virtual network are free to talk to each other.
* Best practice is to create a new virtual network for each app if they are not related
* A container can be attached to multiple virtual networks (much like PCs can have multiple NICs). Or none.
* A container can skip NAT and use the host networking `--net=host` (not recommended).
* There are "network drivers" too...

`docker container port nginx` - Shows port configuration of the container
`docker container inspect --format '{{ .NetworkSettings.IPAddress }}' nginx` - Extracts the IP address of the container on the virtual network

`docker network ls` - Lists virtual networks
`docker network create my_network [--driver ...]` - Create new virtual network (optionally with a driver)
`docker network rm my_network` - Delete virtual network
`docker network inspect bridge` - JSON metadata about the virtual network, for example Subnet, default gateway, etc. Lists containers on it and their MAC addresses too.

`docker container run --rm -d --network my_network nginx` - Run a container on a specific network (`--net` is a shorthand for `--network`)
`docker network --help`
`docker network connect --help`
`docker network connect my_network redis` - Connects container to the network (insert virtual NIC plugged into the given network)
`docker network disconnect my_network redis` -DiscConnects container from the network (remove virtual NIC plugged into the given network)


## Name Resolution

* DNS is needed to reliably identify containers that are coming and going
* Container names are used as DNS names for host resolution on the same virtual network

`docker container exec -it nginx ping nginx2`

* The default "bridge" network does NOT have the DNS server built into it. For that you will need use the `--link` (add a link to another container). Easier to create a new network :).
* Using `docker compose` makes this mush easier by automatically creating virtual networks.

* To create simple round robin DNS name resolution (a poor man's load balancer) use the `--net-allias` option. Container names must be unique, but aliases do not. Aliases will respond to DNS queries in a round-robin fasion.

`docker container run --rm -d --net my_network --net-alias search elasticsearch:2` - (running the first container)
`docker container run --rm -d --net my_network --net-alias search elasticsearch:2` - (running a second container, exact same command!)
`docker container ls` - Notice the ports they use (9200, 9300), we are NOT exposing the ports on the host!
`docker container run --rm --net my_network alpine nslookup search` -- Should list both container IP addresses.
`docker container run --rm --net my_network centos curl -s search:9200` -- Calling elasticsearch by the alias. Repeatedly calling should alternate between the two containers. Check the server's `name` attribute to verify. *THIS DID NOT WORK FOR ME. IT ALWAYS CALLED THE SAME CONTAINER. ONY SWITHED TO THE OTHER ONE WHEN THE FIRST CONTAINER WAS STOPPED.*


## Building Images
