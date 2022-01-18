# Base Conda Environment

This is a Docker image for building a base conda environment

## Docker build

Build using the command below. Make sure to change the platform to whatever you need, e.g., `linux/arm64`, `linux/amd64`, etc...

```bash
cd build/debian
docker build \
    --file=Dockerfile \
    --platform=linux/arm64 \
    --target=conda_base_debian \
    --tag=conda_base_dev:v0.0.0a \
    --build-arg IMAGE_CREATED=2021-11-11T11:11:11Z \
    --build-arg IMAGE_VERSION=v0.0.0a \
    --build-arg CONDA_DIST=Mambaforge \
    .
```

For pushing to docker hub.

The Dockerfile makes use of `buildkit`. To push multiple architectures at a time, you need `buildx`. You may need to set this up before trying to build the image.

```bash
docker buildx install
docker buildx create --platform linux/arm64,linux/amd64 --name=mrbuilder --use
```

```bash
cd build/debian
VERSION=v1.0.0
docker buildx build \
    --file=Dockerfile \
    --push \
    --platform=linux/arm64,linux/amd64 \
    --target=conda_base_debian \
    --tag=iamamutt/conda_base:$VERSION \
    --tag=iamamutt/conda_base:latest \
    --build-arg IMAGE_CREATED=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
    --build-arg IMAGE_VERSION=$VERSION \
    .
```

To remove the installed buildx builder
    
```bash
docker buildx rm mrbuilder
docker buildx uninstall
```

<!-- 

```bash
cd build/debian
docker buildx build \
    --file=Dockerfile \
    --platform=linux/arm64 \
    --output=type=docker \
    --target=conda_base_debian \
    --tag=conda_base_dev:v0.0.0a \
    --build-arg IMAGE_CREATED=2021-11-11T11:11:11Z \
    --build-arg IMAGE_VERSION=v0.0.0a \
    --build-arg CONDA_DIST=Mambaforge \
    .
```

#> git tag -a v1.0.0 -m "GitHub Actions Initial Workflow"
#> git push --tags
#> git push origin v1.0.0

#> git tag -d v1.0.0
#> git push --delete origin v1.0.0
-->
