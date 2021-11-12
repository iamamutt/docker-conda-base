# Base Conda Environment

This is a Docker image for building a base conda environment

## Docker build

The Dockerfile makes use of `buildx`. You may need to set this up before trying to build the image.

```bash
docker buildx install
docker buildx create --platform linux/arm64,linux/arm/v8,linux/amd64 --name=mrbuild --use
```

Then build using the command below. Make sure to change the platform to whatever you need, e.g., `linux/arm64`, `linux/amd64`, etc...

```bash
cd build/debian
docker buildx build \
    --file=Dockerfile \
    --load \
    --platform=linux/arm64 \
    --target=conda_base_debian \
    --tag=conda_base_dev:v0.0.0 \
    --build-arg IMAGE_CREATED=2021-11-11T11:11:11Z \
    --build-arg IMAGE_VERSION=v0.0.0 \
    --build-arg CONDA_DIST=Mambaforge \
    .
```

For pushing to docker hub.

```bash
cd build/debian
VERSION=v1.0.1
docker buildx build \
    --file=Dockerfile \
    --push \
    --platform=linux/amd64,linux/arm64 \
    --target=conda_base_debian \
    --tag=iamamutt/conda_base:$VERSION \
    --tag=iamamutt/conda_base:latest \
    --build-arg IMAGE_CREATED=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
    --build-arg IMAGE_VERSION=$VERSION \
    .
```
