Docker is a virtualization tool used to ease development and deployment of applications. Docker does this via packaging applications with all dependencies and configuration into [[#Image|images]] that can be deployed or run locally easily as [[#Container|containers]]. Originally Docker was developed for linux OS
###### Image
- A packaged application with all the necessary dependencies, configuration, and environment settings, and a start script
- A portable artifact, easily shared and moved around = makes development and deployment more efficient
- Stored in a repository (public/private)
- Think of a class/template for container
###### Container
- A running instance of an image (think of an instance of a class)
- Own isolated environment (achieved via namespaces)
- Has its own virtual file system
- Technically, container is consisted of layers of images stacked on top of each. The advantage of layers is decoupling - in case an image changes, only that changed image is downloaded
	- **Base layer** - mostly linux base image (e.g. alpine:3.10 or different distro) because small in size
	- *A number of intermediate layers/images can reside between base and app image/layer*
	- **Application layer on top** - application image (e.g. postgres:10.10)
###### Dockerfile
- basically a template for building an image (think of class code)
- describes base OS, installed software & dependencies, app code/binary, commands to run upon start
- always called "Dockerfile" and nothing else
- every change to Dockerfile means rebuilding an image because the changes would not reflect in the old image
```dockerfile
FROM alpine:latest

WORKDIR /app

# executes on the HOST env
COPY app.py .

# executes INSIDE the container env
RUN apt-get update && apt-get install -y curl
RUN npm install
RUN pip install flask

# Documents that app uses the port; does NOT open the port - that is done via docker run command
EXPOSE 5000 

ENV FLASK_ENV=production %% Environment variables (runtime) %%

ARG VERSION=1.0 %% Build-time variables %%

# Run as non-root (safer). By default containers run as root
USER appuser

# Data persists outside container. Often handled at runtime
VOLUME /data 

# Fixed startup command (cannot be overriden easily)
# ENTRYPOINT ["python"] 
# CMD ["app.py"]

# Entry point command. CMD with JSON array syntax is preffered in contrast with `CMD python app.py`. Without JSON array syntax docker uses the default shell /bin/sh to run the command. This can be problematic as the docker hides the fact that it's using the shell. It might cause unexpected behavior or errors.
CMD ["python", "app.py"]  
```
###### Docker Compose
- a tool for running multiple containers together (easier)
- most applications have multiple parts such as database, cache, background worker, web app, etc. and they need networking, environment variables, startup order of services or shared volumes etc. - docker compose handles all of that automatically; **everything starts together, connected and configured**
- instead of writing complex commands for running N services like these:
- regarding environment variables; it is probably better to define them externally in a docker-compose file (rather than in Dockerfile). This way we can override them in docker-compose file (if something changes) instead of rebuilding the image (as in the case of env variables in Dockerfile)
```shell
docker run -p 6000:6000 --name postgres -e USER=admin -e PASSWORD=secret -d --net private-network postgres:latest

docker run ...
docker run ...
```
- you write a one file `docker-compose.yml` and execute it `docker compose up`
-  example docker-compose file:
```yaml
services:
  web:
    image: node:18
    ports:
      - "3000:3000" %% or with .env file "${PORT}:3000" super useful %%
    command: npm run dev %% overrides Dockerfile CMD %%

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb 
      
  worker:
    build:
      context: .
      dockerfile: Dockerfile %% build from local Dockerfile %%
    depends_on:
      - db %% ensures container starts, but not that service is ready (retry) %%
    restart: always ( or on-failure, unless-stopped, no) %% restart policy %%
      
volumes:
  - db-data:/var/lib/postgresql/data
```
###### Volumes
Restarting or removing a docker container causes all configuration and data to be gone. For that case, there are container volumes which persist data **outside** containers.
- It is a directory in the host machine's physical file system (managed by docker) and which is mounted into the container's virtual file system
- All data is replicated if changed in either directory (virtual/physical)
- Databases **always** need a volume
- **Host volumes** - you decide where on the host file system the reference is made. Created with `docker run -v /host/dir:/virtual/dir`
- **Anonymous volumes** - you specify just the virtual directory. The reference on the host file system is created and managed by docker automatically. Created with 
  `docker run -v /virtual/dir`
- **Named volumes** - improvement of anonymous volumes. You specify the name of the reference on the host file system. Created with `docker run -v name:/virtual/dir`. Then it is possible to reference that volume by name - you do not have to know the specific path to volume. *Should be used in production and overall*

---

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

# read docker file and create an image with name (and optional tag)
docker build -t <name:tag> .
```



