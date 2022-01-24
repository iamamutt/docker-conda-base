# docker-conda-base

A base docker image for a containerized conda environment

## Usage

### docker run

You can start the container using the defaults:

```bash
docker run --name conda_base_python -itd --init ghcr.io/iamamutt/conda_base:latest dev
```

### docker-compose

You can also setup a custom user, debian packages, and conda environment using the file `docker-compose.yml`.

Download the repository from GitHub

```bash
git clone https://github.com/iamamutt/docker-conda-base.git
cd docker-conda-base
```

You can bind mount any local directory to the container directory `/srv/conda`. The local directory should contain a `apt_requirements.txt` for Debian package installation and a `environment.yml` file for setting up a new conda environment.

For example, using the GitHub repository contents,

1. Edit the content of `./share/apt_requirements.txt` to install custom debian packages, and edit the content of `environment.yml` to add custom conda and pip packages. In `docker-compose.yml`, the volume must be bind mounted from host `./share` to `/srv/conda` in the remote container to detect these files and use them during initialization.

2. Edit the environment variable `NEW_USER_NAME` in `docker-compose.yml` to create a new user.

3. Next, run the following:

```bash
docker-compose up --detach
```

### Extending the base image

You can create a new `Dockerfile` and import the base image to extend a new image. Assuming you have the files `apt_requirements.txt` and `environment.yml` in your build context.

```dockerfile
FROM ghcr.io/iamamutt/conda_base:latest
ARG NEW_USER_NAME=newuser
COPY apt_requirements.txt environment.yml /srv/conda/
RUN init-env
USER ${NEW_USER_NAME}
WORKDIR /home/${NEW_USER_NAME}
ENTRYPOINT [ "conda-run" ]
CMD ["tail", "-f", "/dev/null"]
```

Then build and run the image,

```bash
docker build --tag=myimage:latest --load .
docker run --name conda_base_python -itd --init myimage:latest dev
```

## Environment variables

The following environment variables can be set when starting the container from the build image.

`NEW_USER_NAME` : Name of the new user

`NEW_USER_GROUP` : Name of the new group (default=condauser)

`NEW_USER_UID` :  The id number to set for the new user (default=1000)

`NEW_USER_GID` :  New user's group id (default=1000)

`NEW_USER_SUDO` :  Set to `true` to enable `sudo` for new user (default=`false`)
