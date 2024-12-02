#
# Build container
#

FROM jupyter/base-notebook:2023-05-15 as build

# Maximum jobs for make
ARG MAX_JOBS=4

USER root

RUN apt-get -y -q update \
 && apt-get -y -q upgrade \
 && apt-get -y -q install \
	build-essential \
	cmake \
	git \
	libboost-log1.74-dev \
	libboost-program-options1.74-dev \
	libeigen3-dev \
	libnetcdf-c++4-dev \
	netcdf-bin \
 && rm -rf /var/lib/apt/lists/*

# Catch2 compilation
WORKDIR /build
RUN git clone -b v2.x https://github.com/catchorg/Catch2.git \
 && cd Catch2 \
 && cmake -Bbuild -H. -DBUILD_TESTING=OFF \
 && cmake --build build/ --target install

# Get nextsimdg source from the dedicated branch for test case
WORKDIR /build
RUN git clone -b develop https://github.com/nextsimdg/nextsimdg.git \
 && cd nextsimdg \
 && cmake -DCMAKE_BUILD_TYPE=Release -B build/ \
 && make -j ${MAX_JOBS} -C build


# Micromamba container

FROM mambaorg/micromamba:1.5.8 as micromamba

#
# Final container
#

FROM jupyter/base-notebook:2023-05-15

# Disable announcements
RUN jupyter labextension disable "@jupyterlab/apputils-extension:announcements"

USER root

ARG MAMBA_USER=mambauser
ARG MAMBA_USER_ID=57439
ARG MAMBA_USER_GID=57439
ENV MAMBA_USER=$MAMBA_USER
ENV MAMBA_ROOT_PREFIX="/opt/conda"
ENV MAMBA_EXE="/bin/micromamba"

COPY --from=micromamba "$MAMBA_EXE" "$MAMBA_EXE"
COPY --from=micromamba /usr/local/bin/_activate_current_env.sh /usr/local/bin/_activate_current_env.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_shell.sh /usr/local/bin/_dockerfile_shell.sh
COPY --from=micromamba /usr/local/bin/_entrypoint.sh /usr/local/bin/_entrypoint.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_initialize_user_accounts.sh /usr/local/bin/_dockerfile_initialize_user_accounts.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_setup_root_prefix.sh /usr/local/bin/_dockerfile_setup_root_prefix.sh

RUN /usr/local/bin/_dockerfile_initialize_user_accounts.sh && \
    /usr/local/bin/_dockerfile_setup_root_prefix.sh

USER $MAMBA_USER

SHELL ["/usr/local/bin/_dockerfile_shell.sh"]

ENTRYPOINT ["/usr/local/bin/_entrypoint.sh"]
CMD ["/bin/bash"]

RUN micromamba install --yes --name base --channel conda-forge \
      xarray matplotlib cartopy cmocean numpy netcdf4 dask nbgitpuller && \
    micromamba clean --all --yes
    
USER root

RUN apt-get -y -q update \
 && apt-get -y -q upgrade \
 && apt-get -y -q install \
	bash-completion \
	libnetcdf-c++4-dev \
	libboost-log1.74 \
	libboost-program-options1.74 \
	libeigen3-dev \
	netcdf-bin \
        vim \
	wget \
 	cmake \
	git \
&& rm -rf /var/lib/apt/lists/*

# Copy from build container
COPY --from=build /build/nextsimdg/build/ /opt/nextsimdg
RUN  ln -s /opt/nextsimdg/nextsim /usr/local/bin/

RUN git clone -b develop https://github.com/nextsimdg/nextsimdg.git

# Adding necessary group for SUMMER fs
RUN groupadd -g 10128 pr-sasip \
 && usermod -g 10128 $NB_USER



USER $NB_USER
