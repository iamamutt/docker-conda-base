# syntax=docker/dockerfile:1.3-labs

# global arguments shared across build stages
ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG USERNAME=condauser
ARG GROUPNAME=condauser
ARG USER_UID=1000
ARG USER_GID=1000
ARG TIMEZONE="America/Chicago"

# Miniforge, Mambaforge, Miniforge-pypy3, Mambaforge-pypy3
ARG CONDA_DIST=Mambaforge

## Base conda build stage ==============================================================
FROM debian:11-slim as conda_base_debian_0

# inherit from global args
ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG USERNAME
ARG GROUPNAME
ARG USER_UID
ARG USER_GID
ARG TIMEZONE
ARG CONDA_DIST

LABEL org.opencontainers.image.authors="Joseph Burling"
LABEL org.opencontainers.image.title="${CONDA_DIST}_base_debian_0"
LABEL org.opencontainers.image.created="$IMAGE_CREATED"
LABEL org.opencontainers.image.version="$IMAGE_VERSION"
LABEL org.opencontainers.image.description="A debian-based conda python environment (install)"

ENV CONDA_DIR=/opt/local/conda
ENV PATH=${CONDA_DIR}/bin:$PATH

# set default shell to exit on first error
SHELL [ "/bin/sh", "-e", "-c" ]

# Install linux dependencies -----------------------------------------------------------
ENV TZ=${TIMEZONE}
RUN <<-EOF
	apt-get update
	apt-get -qq install -y apt-utils
	export DEBIAN_FRONTEND=noninteractive
	apt-get -qq install -y locales wget sudo
	localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
	ln -snf /usr/share/zoneinfo/${TZ} /etc/localtime
	echo ${TZ} > /etc/timezone
	apt-get autoremove -y
	apt-get clean -y
	rm -rf /var/lib/apt/lists/*
EOF
ENV LANG=en_US.utf8
ENV LC_ALL=en_US.utf8

# Add directories, user/group ----------------------------------------------------------
RUN	<<-EOF
	groupadd --gid ${USER_GID} --force ${GROUPNAME}
	umask a+rwx,o-wx
	mkdir -p ${CONDA_DIR}
	cd ${CONDA_DIR}
	chown -R :${USER_GID} .*
	chgrp --no-dereference ${USER_GID} /etc/profile.d /var/log
	chmod -R 774 /etc/profile.d /var/log
	echo ". '/etc/profile.d/conda.sh'\nconda activate base" >> /etc/skel/.bashrc
	useradd --uid ${USER_UID} --gid ${USER_GID} \
		--create-home --system --groups users,sudo --shell /bin/bash ${USERNAME}
	passwd -d ${USERNAME}
	echo '${USERNAME} ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
	install -m 771 /dev/null /usr/local/bin/conda-run
	chown :${USER_GID} /usr/local/bin/conda-run
EOF

# Entrypoint to activate specific conda environment ------------------------------------
RUN cat <<-'EOF' >> /usr/local/bin/conda-run
	#! /bin/bash --login
	[ -z "$CONDA_ENV_USER" ] && CONDA_ENV_USER=base
	set -eo pipefail
	conda activate $CONDA_ENV_USER
	exec -l "$@"
EOF

# Install conda as user ----------------------------------------------------------------
USER ${USER_UID}:${USER_GID}
WORKDIR /home/${USERNAME}
RUN <<-EOF
	URL=https://github.com/conda-forge/miniforge/releases/latest/download/${CONDA_DIST}-$(uname)-$(uname -m).sh
	wget --no-hsts --quiet -O ~/conda_install.sh ${URL}
	chmod +x ~/conda_install.sh
	~/conda_install.sh -vbup ${CONDA_DIR}
	rm ~/conda_install.sh
	find ${CONDA_DIR} -follow -type f -name '*.a' -delete
	find ${CONDA_DIR} -follow -type f -name '*.pyc' -delete
	find ${CONDA_DIR} -follow -type f -name '*.js.map' -delete
	ln -s ${CONDA_DIR}/etc/profile.d/conda.sh /etc/profile.d/conda.sh
EOF

# switch to bash as default shell for the next RUN commands, conda already in PATH
SHELL [ "/bin/bash", "-ec" ]

# configure and update base conda
RUN <<-EOF
	conda install -qy conda-build
	conda config --add channels conda-forge
	conda config --append channels defaults
	conda config --set pip_interop_enabled true
	conda clean -yiqaf
EOF

## Target stage ========================================================================
FROM scratch as conda_base_debian
COPY --from=conda_base_debian_0 / /
ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG CONDA_DIST
ARG USERNAME
ARG USER_UID
ARG USER_GID
ARG TIMEZONE
LABEL org.opencontainers.image.authors="Joseph Burling"
LABEL org.opencontainers.image.title="${CONDA_DIST}_base_debian_build"
LABEL org.opencontainers.image.created="$IMAGE_CREATED"
LABEL org.opencontainers.image.version="$IMAGE_VERSION"
LABEL org.opencontainers.image.description="A debian-based conda python environment"
ENV TZ=${TIMEZONE}
ENV LANG=en_US.utf8
ENV LC_ALL=en_US.utf8
ENV CONDA_ENV_USER=base
USER ${USER_UID}:${USER_GID}
WORKDIR /home/${USERNAME}
ENTRYPOINT [ "conda-run" ]
CMD [ "tail", "-f", "/dev/null" ]
