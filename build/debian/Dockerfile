# syntax=docker/dockerfile:1.3-labs

# global arguments shared across build stages
ARG USER_NAME=condauser
ARG USER_GROUP=condauser
ARG USER_UID=999
ARG USER_GID=999
ARG TIMEZONE="America/Chicago"

# Miniforge, Mambaforge, Miniforge-pypy3, Mambaforge-pypy3
ARG CONDA_DIST=Mambaforge
ARG CONDA_ROOT_DIR=/opt/local/conda

## Base conda build stage ==============================================================
FROM debian:11-slim as conda_base_debian_0

# inherit from global args
ARG USER_NAME
ARG USER_GROUP
ARG USER_UID
ARG USER_GID
ARG TIMEZONE
ARG CONDA_DIST
ARG CONDA_ROOT_DIR

LABEL org.opencontainers.image.authors "Joseph Burling"
LABEL org.opencontainers.image.title "${CONDA_DIST}_base_debian_0"
LABEL org.opencontainers.image.description "A debian-based conda python environment (install)"

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

# Add default group --------------------------------------------------------------------
RUN	<<-EOF
	rm /var/log/lastlog /var/log/faillog
	ln -s /dev/null /var/log/lastlog
	ln -s /dev/null /var/log/faillog
	groupadd --gid ${USER_GID} --force ${USER_GROUP}
	chgrp --no-dereference ${USER_GID} /etc/profile.d /var/log
	chmod -R 774 /etc/profile.d /var/log
	rm /var/log/lastlog /var/log/faillog
	touch /var/log/lastlog
	touch /var/log/faillog
EOF

# Skeleton files for new users and root startup scripts --------------------------------
RUN	<<-EOF
	cd /etc/skel
	echo "\n# >>> conda_base_debian >>>" | tee -a \
		.bashrc .bash_profile /root/.bashrc /root/.bash_profile /root/.profile
	echo ". '/etc/profile.d/conda.sh'" | tee -a \
		.bashrc /root/.bashrc
	echo "conda activate base" | tee -a \
		.bashrc .bash_profile /root/.bashrc /root/.bash_profile /root/.profile
	echo "# <<< conda_base_debian <<<" | tee -a \
		.bashrc .bash_profile /root/.bashrc /root/.bash_profile /root/.profile
	mkdir -p .config .cache .local/share .local/state .local/bin
	chmod -R 600 .config .cache .local
EOF

# Add directories / files --------------------------------------------------------------

# switch to default group for future permissions
USER 0:${USER_GID}

# switch to bash as default shell for the next RUN commands, conda already in PATH
SHELL [ "/bin/bash", "-ec" ]

# copy over initialization scripts
COPY --chown=0:${USER_GID} \
	./init-env ./init-apt-deps ./init-new-user ./init-conda-env \
	/usr/local/bin/

# change permissions for init scripts and conda dirs
RUN	<<-EOF
	umask a+rwx,o-wx
	chmod 770 \
		/usr/local/bin/init-env \
		/usr/local/bin/init-apt-deps \
		/usr/local/bin/init-new-user \
		/usr/local/bin/init-conda-env
	install -m 770 /dev/null /usr/local/bin/conda-run
	chown :${USER_GID} /usr/local/bin/conda-run
	mkdir -p ${CONDA_ROOT_DIR}/envs ${CONDA_ROOT_DIR}/pkgs /srv/conda
	cd ${CONDA_ROOT_DIR}
	chown -R :${USER_GID} .*
	install -m 644 /dev/null /srv/conda/apt_requirements.txt
	install -m 644 /dev/null /srv/conda/environment.yml
	chown :${USER_GID} /srv/conda /srv/conda/*
EOF

# Entrypoint to activate specific conda environment ------------------------------------
RUN cat <<-'EOF' >> /usr/local/bin/conda-run
	#! /bin/bash --login
	echo ">> $0 $* --"
	[ -z "$CONDA_ENV_USER" ] && CONDA_ENV_USER=base
	set -eo pipefail
	conda activate $CONDA_ENV_USER
	exec $@
EOF

# Default apt-get dependencies ---------------------------------------------------------
RUN cat <<-EOF >> /srv/conda/apt_requirements.txt
	bash
EOF

# Default conda environment files ------------------------------------------------------
RUN cat <<-EOF >> /srv/conda/environment.yml
	name: default
	dependencies:
	  - python
	  - pip
EOF

# Install conda ------------------------------------------------------------------------
WORKDIR /srv/conda
ENV PATH=${CONDA_ROOT_DIR}/bin:$PATH
RUN <<-EOF
	URL=https://github.com/conda-forge/miniforge/releases/latest/download/${CONDA_DIST}-$(uname)-$(uname -m).sh
	wget --no-hsts --quiet -O /srv/conda/conda_install.sh ${URL}
	chmod +x /srv/conda/conda_install.sh
	/srv/conda/conda_install.sh -vbup ${CONDA_ROOT_DIR}
	ln -s ${CONDA_ROOT_DIR}/etc/profile.d/conda.sh /etc/profile.d/conda.sh
	conda config --system --prepend envs_dirs ${CONDA_ROOT_DIR}/envs
	conda config --system --prepend pkgs_dirs ${CONDA_ROOT_DIR}/pkgs
	conda config --system --add channels conda-forge
	conda config --system --append channels defaults
	conda config --system --set pip_interop_enabled true
	cp -f ${CONDA_ROOT_DIR}/.condarc /etc/skel/.condarc
	cp -f ${CONDA_ROOT_DIR}/.condarc /root/.condarc
	conda install -qy conda-build
	conda clean -yiqaf
	rm /srv/conda/conda_install.sh
	find ${CONDA_ROOT_DIR} -follow -type f -name '*.a' -delete
	find ${CONDA_ROOT_DIR} -follow -type f -name '*.pyc' -delete
	find ${CONDA_ROOT_DIR} -follow -type f -name '*.js.map' -delete
	chmod 770 -R ${CONDA_ROOT_DIR}/envs ${CONDA_ROOT_DIR}/pkgs
	chmod a+rwX,o-w ${CONDA_ROOT_DIR}/*
EOF

# Add default user ---------------------------------------------------------------------
RUN	<<-EOF
	export IMAGE_DEFAULT_USER_NAME=${USER_NAME}
	export IMAGE_DEFAULT_USER_GROUP=${USER_GROUP}
	init-new-user "${USER_NAME}" "${USER_GROUP}" ${USER_UID} ${USER_GID} true
EOF

## Target stage ========================================================================
FROM scratch as conda_base_debian
COPY --from=conda_base_debian_0 / /

ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG CONDA_DIST
LABEL org.opencontainers.image.authors "Joseph Burling"
LABEL org.opencontainers.image.title "${CONDA_DIST}_base_debian_build"
LABEL org.opencontainers.image.description "A debian-based conda python environment"
LABEL org.opencontainers.image.version "$IMAGE_VERSION"
LABEL org.opencontainers.image.created "$IMAGE_CREATED"

ARG USER_NAME
ARG USER_GROUP
ARG CONDA_ROOT_DIR
ARG TIMEZONE
ENV TZ=${TIMEZONE}
ENV LANG=en_US.utf8
ENV LC_ALL=en_US.utf8
ENV IMAGE_DEFAULT_USER_NAME=${USER_NAME}
ENV IMAGE_DEFAULT_USER_GROUP=${USER_GROUP}
ENV CONDA_DIR=${CONDA_ROOT_DIR}
ENV CONDA_ROOT=${CONDA_ROOT_DIR}
ENV CONDA_ENV_USER=

USER root:${USER_GROUP}
WORKDIR /srv/conda
ENTRYPOINT [ "init-env" ]
