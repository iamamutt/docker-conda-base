# docker-conda-base

A base docker image for a containerized conda environment

## Installation 

Download the repository from GitHub

```bash
git clone https://github.com/iamamutt/docker-conda-base.git
cd docker-conda-base
```

### docker run

You can start the container using the defaults:

```bash
docker run --name conda_base_python -itd --init iamamutt/conda_base:latest
```

### docker-compose 

You can also setup a custom user, debian packages, and conda environment using the file `docker-compose.yml`.

1. Edit the contents of `./share/apt_requirements.txt` to install custom debian packages, and edit the content of `environment.yml` to add custom conda and pip packages. The volume must be bind mounted from host directory `./share` to `/srv/conda` in the remote container to detect these files and use them during initialization. 

2. Edit the environment variables `USER_NAME` and `USER_ID` in `docker-compose.yml` to create a new user.

3. Next, run the following:

```bash
docker-compose up --detach
```
