# docker-conda-base

A base docker image for a containerized conda environment

https://hub.docker.com/repository/docker/iamamutt/conda_base

## Installation 

Download the repository from GitHub

```bash
git clone https://github.com/iamamutt/docker-conda-base.git
cd docker-conda-base
```

### docker run

You can start the container using the defaults:

```bash
docker run --name conda_base_python -itd --init iamamutt/conda_base:latest dev
```

### docker-compose 

You can also setup a custom user, debian packages, and conda environment using the file `docker-compose.yml`.

1. Edit the contents of `./share/apt_requirements.txt` to install custom debian packages, and edit the content of `environment.yml` to add custom conda and pip packages. The volume must be bind mounted from host directory `./share` to `/srv/conda` in the remote container to detect these files and use them during initialization. 

2. Edit the environment variables `USER_NAME` and `USER_UID` in `docker-compose.yml` to create a new user.

3. Next, run the following:

```bash
docker-compose up --detach
```

## Extending the base image

You can create a new `Dockerfile` and import the base image to extend a new image. Assuming you have the files `apt_requirements.txt` and `environment.yml` in your build context. 

```dockerfile
FROM iamamutt/conda_base:latest
ARG USER_NAME=newuser
COPY apt_requirements.txt environment.yml ./
RUN init-env
USER ${USER_NAME}
WORKDIR /home/${USER_NAME}
ENTRYPOINT [ "conda-run" ]
CMD ["tail", "-f", "/dev/null"]
```

Then build the image,

```bash
docker build --tag=myimage:latest --load .
```
