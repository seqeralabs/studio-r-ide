# Copyright 2023-2026, Seqera Labs
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG CONNECT_CLIENT_VERSION=0.14.0
FROM public.cr.seqera.io/platform/connect-client:${CONNECT_CLIENT_VERSION} AS connect

FROM mambaorg/micromamba:2.5.0 as micromamba

FROM ubuntu:24.04

ARG CONNECT_CLIENT_VERSION
LABEL io.seqera.connect.version="${CONNECT_CLIENT_VERSION}"

ARG TARGETARCH

ARG RSTUDIO_VERSION=2026.01.2
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

# ubuntu:24.04 ships /etc/localtime as a symlink to /usr/share/zoneinfo//UTC
# (double-slash), which makes R's Sys.timezone() return the invalid Olson
# name "/UTC". Rewrite to a clean target.
RUN ln -sf /usr/share/zoneinfo/Etc/UTC /etc/localtime && echo "Etc/UTC" > /etc/timezone

## install rstudio server
RUN if [ ${TARGETARCH} = "arm64" ]; then \
      wget https://github.com/seqeralabs/r-ide/releases/download/seqera-v2026.01.2%2B418/arm64.deb -O ride.deb; \
    elif [ ${TARGETARCH} = "amd64" ]; then \
      wget https://github.com/seqeralabs/r-ide/releases/download/seqera-v2026.01.2%2B418/amd64.deb -O ride.deb; \
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

COPY --chown=$MAMBA_USER:$MAMBA_USER ./env.yaml /tmp/env.yaml
RUN micromamba install --yes --name base --file /tmp/env.yaml && \
    micromamba clean --all --yes && \
    rm -f /tmp/env.yaml

# Default CRAN repo to Posit Public Package Manager (noble matches the
# ubuntu:24.04 base - update if that bumps). P3M serves prebuilt binaries
# only when the HTTP User-Agent identifies a Linux R client; R's default
# UA omits that, so we set HTTPUserAgent explicitly. Without it P3M falls
# back to source tarballs and install.packages() ends up compiling.
RUN { \
      echo 'options(repos = c(CRAN = "https://packagemanager.posit.co/cran/__linux__/noble/latest"))'; \
      echo 'options(HTTPUserAgent = sprintf("R/%s R (%s)", getRversion(), paste(getRversion(), R.version$platform, R.version$arch, R.version$os)))'; \
    } >> /opt/conda/lib/R/etc/Rprofile.site

# Setup NGINX for path routing
COPY ./nginx.conf.template /etc/nginx/nginx.conf.template

# Setup RStudio configuration
COPY ./init /init
COPY ./rserver.conf /etc/rstudio/rserver.conf
COPY ./rsession.conf /etc/rstudio/rsession.conf
COPY ./logging.conf /etc/rstudio/logging.conf
COPY ./keybindings/editor_bindings.json /etc/rstudio/keybindings/editor_bindings.json
COPY ./keybindings/rstudio_bindings.json /etc/rstudio/keybindings/rstudio_bindings.json
COPY ./keybindings/addins.json /etc/rstudio/keybindings/addins.json
COPY ./Renviron /root/rstudio/.Renviron

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