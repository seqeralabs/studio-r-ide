ARG CONNECT_CLIENT_VERSION

FROM public.cr.seqera.io/platform/connect-client:0.11 AS connect

FROM mambaorg/micromamba:2.5.0 as micromamba

FROM ubuntu:24.04

ARG CONNECT_CLIENT_VERSION
LABEL io.seqera.connect.version="${CONNECT_CLIENT_VERSION}"

ARG TARGETARCH

ARG RSTUDIO_VERSION=2024.12.1-563
ARG MAMBA_USER=root
ARG MAMBA_USER_ID=0
ARG MAMBA_USER_GID=0
ARG MAMBA_DOCKERFILE_ACTIVATE=1
ENV MAMBA_USER=$MAMBA_USER
ENV MAMBA_ROOT_PREFIX="/opt/conda"
ENV MAMBA_EXE="/bin/micromamba"
USER $MAMBA_USER
ENV USER=$MAMBA_USER

# install required packages
RUN apt-get update --yes && apt-get install --yes --no-install-recommends \
    ca-certificates \
    wget \
    curl \
    git \
    sudo \
    libssl-dev \
    libclang-dev \
    lsb-release \
    psmisc \
    gdebi-core \
    gettext-base \
    nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

## install rstudio server
RUN if [ ${TARGETARCH} = "arm64" ]; then \
      wget https://github.com/seqeralabs/r-ide/releases/download/seqera-v2025.05.0%2B496/arm64.deb -O ride.deb; \
    elif [ ${TARGETARCH} = "amd64" ]; then \
      wget https://github.com/seqeralabs/r-ide/releases/download/seqera-v2025.05.0%2B496/amd64.deb -O ride.deb; \
    else \
      echo "Unsupported architecture: ${TARGETARCH}" && exit 1; \
    fi && \
    gdebi -n ride.deb && \
    rm ride.deb

# micromamba setup
COPY --from=micromamba "$MAMBA_EXE" "$MAMBA_EXE"
COPY --from=micromamba /usr/local/bin/_activate_current_env.sh /usr/local/bin/_activate_current_env.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_shell.sh /usr/local/bin/_dockerfile_shell.sh
COPY --from=micromamba /usr/local/bin/_entrypoint.sh /usr/local/bin/_entrypoint.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_initialize_user_accounts.sh /usr/local/bin/_dockerfile_initialize_user_accounts.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_setup_root_prefix.sh /usr/local/bin/_dockerfile_setup_root_prefix.sh

RUN micromamba shell init --shell bash && \
    /usr/local/bin/_dockerfile_initialize_user_accounts.sh && \
    /usr/local/bin/_dockerfile_setup_root_prefix.sh

COPY --chown=$MAMBA_USER:$MAMBA_USER .seqera/env.yaml /tmp/env.yaml
RUN micromamba install --yes --name base --file /tmp/env.yaml && \
    micromamba clean --all --yes && \
    rm -f /tmp/env.yaml

# Setup NGINX for path routing
COPY .seqera/nginx.conf.template /etc/nginx/nginx.conf.template

# Setup RStudio configuration
COPY .seqera/init /init
COPY .seqera/rserver.conf /etc/rstudio/rserver.conf
COPY .seqera/rsession.conf /etc/rstudio/rsession.conf
COPY .seqera/logging.conf /etc/rstudio/logging.conf
COPY .seqera/keybindings/editor_bindings.json /etc/rstudio/keybindings/editor_bindings.json
COPY .seqera/keybindings/rstudio_bindings.json /etc/rstudio/keybindings/rstudio_bindings.json
COPY .seqera/keybindings/addins.json /etc/rstudio/keybindings/addins.json
COPY .seqera/Renviron /root/rstudio/.Renviron

RUN chmod +x /init

# configure directories and shell
RUN mkdir -p /workspace
SHELL ["/bin/bash", "-c"]

# Add connect binary
COPY --from=connect /usr/bin/connect-client /usr/bin/connect-client
# Install connect dependencies
RUN /usr/bin/connect-client --install
# Optionally set tool identifier for metrics tracking purposes
RUN /usr/bin/connect-client --setToolIdentifier ride

ENTRYPOINT ["/usr/bin/connect-client", "--entrypoint"]
CMD ["/usr/bin/bash", "-c", "/init"]