Docker is a virtualization tool used to ease development and deployment of applications. Docker does this via packaging applications with all dependencies and configuration into [[#Image|images]] that can be deployed or run locally easily as [[#Container|containers]]. Originally Docker was developed for linux OS
###### Image
- A packaged application with all the necessary dependencies, configuration, and environment settings, and a start script
- A portable artifact, easily shared and moved around = makes development and deployment more efficient
- Stored in a repository (public/private)
###### Container
- A running instance of an image (or a running environment for image)
- Own isolated environment (achieved via namespaces)
- Has its own virtual file system
- Technically, container is consisted of layers of images stacked on top of each. The advantage of layers is decoupling - in case an image changes, only that changed image is downloaded
	- **Base layer** - mostly linux base image (e.g. alpine:3.10 or different distro) because small in size
	- *A number of intermediate layers/images can reside between base and app image/layer*
	- **Application layer on top** - application image (e.g. postgres:10.10)
#### Before & After Docker
##### App Development BEFORE Containers
- Every developer that wants to work on some app, would need to install manually each service (that the app uses) on the OS directly (e.g. PostgreSQL + Redis services)
- The installation process differs with OS
- The installation process usually consists of many steps where something could go wrong
- The supervision of human operator is necessary = prone to error
- Multiple versions of the same app can be run concurrently on local environment but with a high risk of conflicts
##### App Deployment BEFORE Containers
- Developers would produce app artifacts (e.g. binary, jar, etc.) with a set of deployment instructions (how to install and configure artifacts on the server)
- DevOps team would then handle the deployment and environment configuration on the server.
- Everything is installed and configured directly on the server's OS which can lead to dependency conflicts
- Everything depends on correct and complete deployment instructions written by HUMANS = prone to errors
##### App Development AFTER Containers
- With containers, developer does not need to install services directly on the OS. Instead he just starts a container which is basically a packaged application with all dependencies and configuration - represents its own isolated environment.
- The whole process shrinks to just one command in terminal per service VS many installation steps per one service. 
- Due to isolation properties of containers, it's possible to run multiple versions of the same app concurrently without conflicts on the local environment
##### App Deployment AFTER Containers
- Development works together with DevOps to package the application in the container
- No configuration needed on the server, just the Docker Runtime
- The deployment consists of running a docker command which pulls the image from repository and runs it 
#### Docker vs. Virtual Machines
Operating system consists of two main layers:
- OS Kernel layer
- Applications layer
Kernel is the part that communicates with HW and applications run on kernel layer. OS Kernel thus acts as a middle man between HW and applications.
##### Docker
- Virtualizes the **application** layer of the OS and uses the OS kernel of the **host** because the containers does not have their own kernel
- Because of that, docker images are much smaller in size (megabytes)
- Docker containers start and run much faster
- Docker containers can run only on linux host OS = compatible only with linux distros because linux based images (majority) cannot run on different than linux kernel (at least not directly)
- With Docker Desktop it is possible to run linux based docker images on different OS (e.g. Windows, Mac) - it is possible using a **hypervisor layer** with a lightweight linux distro that provides a linux kernel
##### Virtual Machine
- Virtualizes the **application** layer AND **OS kernel** layer = **the whole OS**
- Does not use the host kernel, it boots up its own kernel
- Because of that, VM images are huge in size (gigabytes)
- VMs start much slower because of booting up its own kernel + applications on top of it
- VM can run any OS on any hosting OS = compatible with all OS
#### Container-Host Ports
- Multiple containers can run on local host machine machine (thanks to namespaces) with different container ports
- The container has its own set of ports as well as the local machine has its own
- In order to communicate with container through local host machine, there needs to be established a **binding port**:![[Pasted image 20260106173211.png]]
- The host port must be unique for each container, otherwise there will be a conflict and container cannot start
- There can be two containers with the same port (see picture)
#### Container Volumes
Restarting a docker container causes all configuration and data to be gone. For data persistence after container exits, there are container volumes.
*TODO*
#### Commands
```shell
# list all running containers
docker ps

# list all containers including stopped ones (--all)
docker ps -a

# list all local images
docker images

# run image with version (pulls the image if not available locally)
docker run <image:version>

# run image in a detached mode (in background)
docker run -d <image>

# run image with binding port (host->container port)
docker run -p <host_port>:<container_port> <image>

# run image with a custom name
docker run --name <container> <image>

# run image in an existing docker network
docker run --net <network> <image>

# run image with an environment variable
docker run -e <KEY=VALUE> <image>

# start or stop the existing container
docker start|stop <container>

# open a shell or bash inside a running container
docker exec -it <container> sh|bash

# open logs of a container
docker logs <container>

# remove a stopped container
docker rm <container>

# view resource usage stats
docker container stats

# list all docker private networks
docker network ls

# create a new docker network
docker network create <name>
```



