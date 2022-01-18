# syntax=docker/dockerfile:1.3-labs

# global arguments shared across build stages
ARG USER_NAME=condauser
ARG USER_GROUP=condauser
ARG USER_UID=999
ARG USER_GID=999
ARG TIMEZONE="America/Chicago"

# Miniforge, Mambaforge, Miniforge-pypy3, Mambaforge-pypy3
ARG CONDA_DIST=Mambaforge

## Base conda build stage ==============================================================
FROM debian:11-slim as conda_base_debian_0

# inherit from global args
ARG USER_NAME
ARG USER_GROUP
ARG USER_UID
ARG USER_GID
ARG TIMEZONE
ARG CONDA_DIST

LABEL org.opencontainers.image.authors "Joseph Burling"
LABEL org.opencontainers.image.title "${CONDA_DIST}_base_debian_0"
LABEL org.opencontainers.image.description "A debian-based conda python environment (install)"

ENV CONDA_DIR=/opt/local/conda
ENV PATH=${CONDA_DIR}/bin:$PATH

# set default shell to exit on first error
SHELL [ "/bin/sh", "-e", "-c" ]

# Install linux dependencies -----------------------------------------------------------
ENV TZ=${TIMEZONE}
RUN <<-EOF
	mkdir -p /usr/share/man/man1
	echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
	apt-get update
	apt-get -qq install -y apt-utils locales wget ca-certificates sudo
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
	groupadd --gid ${USER_GID} --force ${USER_GROUP}
	umask a+rwx,o-wx
	mkdir -p ${CONDA_DIR} /srv/conda
	cd ${CONDA_DIR}
	chown -R :${USER_GID} .*
	chgrp --no-dereference ${USER_GID} /etc/profile.d /var/log
	chmod -R 774 /etc/profile.d /var/log
	echo ". '/etc/profile.d/conda.sh'\nconda activate base" |
		tee -a /etc/skel/.bashrc /root/.bashrc
	useradd --uid ${USER_UID} --gid ${USER_GID} \
		--create-home --groups users,sudo --shell /bin/bash ${USER_NAME}
	passwd -qd ${USER_NAME}
	echo '${USER_NAME} ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
	install -m 770 /dev/null /usr/local/bin/conda-run
	install -m 644 /dev/null /srv/conda/apt_requirements.txt
	install -m 644 /dev/null /srv/conda/environment.yml
	chown :${USER_GID} /usr/local/bin/conda-run /srv/conda /srv/conda/*
EOF

# Entrypoint to activate specific conda environment ------------------------------------
RUN cat <<-'EOF' >> /usr/local/bin/conda-run
	#! /bin/bash --login
	[ -z "$CONDA_ENV_USER" ] && CONDA_ENV_USER=base
	set -eo pipefail
	conda activate $CONDA_ENV_USER
	exec -l "$@"
EOF

# Default apt-get dependencies and conda environment files -----------------------------
RUN cat <<-EOF >> /srv/conda/apt_requirements.txt
	bash
EOF

RUN cat <<-EOF >> /srv/conda/environment.yml
	name: default
	dependencies:
	  - python
	  - pip
EOF

COPY --chown=0:${USER_GID} \
	./init-env ./init-apt-deps ./init-new-user ./init-conda-env \
	/usr/local/bin/
RUN chmod 770 \
	/usr/local/bin/init-env \
	/usr/local/bin/init-apt-deps \
	/usr/local/bin/init-new-user \
	/usr/local/bin/init-conda-env

# Install conda as user ----------------------------------------------------------------
USER ${USER_UID}:${USER_GID}
WORKDIR /srv/conda

# switch to bash as default shell for the next RUN commands, conda already in PATH
SHELL [ "/bin/bash", "-ec" ]

RUN <<-EOF
	URL=https://github.com/conda-forge/miniforge/releases/latest/download/${CONDA_DIST}-$(uname)-$(uname -m).sh
	wget --no-hsts --quiet -O /srv/conda/conda_install.sh ${URL}
	chmod +x /srv/conda/conda_install.sh
	/srv/conda/conda_install.sh -vbup ${CONDA_DIR}
	rm /srv/conda/conda_install.sh
	find ${CONDA_DIR} -follow -type f -name '*.a' -delete
	find ${CONDA_DIR} -follow -type f -name '*.pyc' -delete
	find ${CONDA_DIR} -follow -type f -name '*.js.map' -delete
	ln -s ${CONDA_DIR}/etc/profile.d/conda.sh /etc/profile.d/conda.sh
	conda install -qy conda-build
	conda config --add channels conda-forge
	conda config --append channels defaults
	conda config --set pip_interop_enabled true
	conda clean -yiqaf
EOF

## Target stage ========================================================================
FROM scratch as conda_base_debian
ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG CONDA_DIST
LABEL org.opencontainers.image.authors "Joseph Burling"
LABEL org.opencontainers.image.title "${CONDA_DIST}_base_debian_build"
LABEL org.opencontainers.image.description "A debian-based conda python environment"
LABEL org.opencontainers.image.version "$IMAGE_VERSION"
LABEL org.opencontainers.image.created "$IMAGE_CREATED"

COPY --from=conda_base_debian_0 / /
ARG USER_NAME
ARG USER_GROUP
ARG TIMEZONE
ENV TZ=${TIMEZONE}
ENV LANG=en_US.utf8
ENV LC_ALL=en_US.utf8
ENV IMAGE_DEFAULT_USER=${USER_NAME}
ENV IMAGE_DEFAULT_GROUP=${USER_GROUP}
ENV CONDA_DIR=/opt/local/conda
ENV CONDA_ENV_USER=
USER root:${USER_GROUP}
WORKDIR /srv/conda
ENTRYPOINT [ "init-env" ]
