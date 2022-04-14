# syntax=docker/dockerfile:1

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
LABEL org.opencontainers.image.title "${CONDA_DIST}_base_debian_pre"
LABEL org.opencontainers.image.description "A debian-based conda-python environment"

# set default shell and to exit on first error
SHELL [ "/bin/bash", "-ec" ]

# Install linux dependencies -----------------------------------------------------------
ENV TZ=${TIMEZONE}
RUN <<-EOF
	mkdir -p -m 6775 /usr/share/man/man1
	echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
	apt-get update
	apt-get -qq install --no-install-recommends -y \
		apt-utils locales wget ca-certificates sudo
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
	rm /var/log/lastlog /var/log/faillog
	touch /var/log/lastlog
	touch /var/log/faillog
	chmod -R 775 /var/log
EOF

# Edit shell config files to find conda for new users and root -------------------------
RUN	<<-EOF
	cfg_n=(/root/.bashrc /etc/skel/.bashrc)
	cfg_l=(/root/.profile /root/.bash_profile /etc/skel/.bash_profile)
	echo -e "\n# >>> conda_base_debian >>>" | tee -a "${cfg_l[@]}" "${cfg_n[@]}" >/dev/null
	echo ". '/etc/profile.d/conda.sh'" | tee -a "${cfg_n[@]}" >/dev/null
	echo "# <<< conda_base_debian <<<" | tee -a "${cfg_l[@]}" "${cfg_n[@]}" >/dev/null
	mkdir -p .config .cache .local/share .local/state .local/bin
EOF

# Add directories / files --------------------------------------------------------------

# switch to default group for future permissions
USER 0:${USER_GID}

# copy over initialization scripts
COPY --chown=0:${USER_GID} ./scripts/ /usr/local/bin/

# change permissions for init scripts and conda dirs
RUN	<<-EOF
	chmod -R 775 /usr/local/bin
	mkdir -p -m 775 /srv/conda
	install -m 664 /dev/null /srv/conda/apt_requirements.txt
	install -m 664 /dev/null /srv/conda/environment.yml
	chown -R :${USER_GID} /srv
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
	mkdir -p ${CONDA_ROOT_DIR}
	chmod 2775 ${CONDA_ROOT_DIR}
	URL=https://github.com/conda-forge/miniforge/releases/latest/download/${CONDA_DIST}-$(uname)-$(uname -m).sh
	wget --no-hsts --quiet -O /srv/conda/conda_install.sh ${URL}
	chmod +x /srv/conda/conda_install.sh
	/srv/conda/conda_install.sh -vbup ${CONDA_ROOT_DIR}
	rm /srv/conda/conda_install.sh
	conda install -qy conda-build
EOF

# Post install steps -------------------------------------------------------------------
RUN <<-EOF
	conda clean -yiqaf
	chmod -R 755 ${CONDA_ROOT_DIR}/etc/profile.d
	ln -s ${CONDA_ROOT_DIR}/etc/profile.d/conda.sh /etc/profile.d/conda.sh
	conda config --system --prepend envs_dirs ${CONDA_ROOT_DIR}/envs
	conda config --system --prepend pkgs_dirs ${CONDA_ROOT_DIR}/pkgs
	conda config --system --add channels conda-forge
	conda config --system --append channels defaults
	conda config --system --set pip_interop_enabled true
	cd ${CONDA_ROOT_DIR}
	cp -f ./.condarc /etc/skel/.condarc
	cp -f ./.condarc /root/.condarc
	find . -follow -type f -name '*.a' -delete
	find . -follow -type f -name '*.pyc' -delete
	find . -follow -type f -name '*.js.map' -delete
	mkdir -p envs pkgs
	chown 0:${USER_GID} . ./* ./.*
	chmod a+rwX,o-w,ug+s,+t,-t . ./* ./.*
EOF

# Add default user ---------------------------------------------------------------------
RUN	<<-EOF
	export IMAGE_DEFAULT_USER_NAME=${USER_NAME}
	export IMAGE_DEFAULT_USER_GROUP=${USER_GROUP}
	init-new-user "${USER_NAME}" "${USER_GROUP}" ${USER_UID} ${USER_GID} true
	rm -rf /tmp/*
EOF

## Target stage ========================================================================
FROM scratch as conda_base_debian
COPY --from=conda_base_debian_0 / /

ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG CONDA_DIST
LABEL org.opencontainers.image.authors "Joseph Burling"
LABEL org.opencontainers.image.title "${CONDA_DIST}_base_debian_target"
LABEL org.opencontainers.image.description "A debian-based conda-python environment"
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
