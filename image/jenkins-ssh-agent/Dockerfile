# Defaults
ARG JENKINS_SSH_DOCKERHUB_IMAGE_NAME=${JENKINS_SSH_DOCKERHUB_IMAGE_NAME:-"jenkins/ssh-agent"}
ARG JENKINS_SSH_DOCKERHUB_IMAGE_TAG=${JENKINS_SSH_DOCKERHUB_IMAGE_TAG:-"latest-jdk21"}

FROM ${JENKINS_SSH_DOCKERHUB_IMAGE_NAME}:${JENKINS_SSH_DOCKERHUB_IMAGE_TAG} AS ssh-agent

ARG DEBIAN_FRONTEND=noninteractive
# OPM url for download (in .env file)
ARG OPM_URL
# Set locales defaults, if no args
ARG LANG
ARG LC_ALL
ARG LOCALE_GEN
ARG LANG=${LANG:-"ru_RU.UTF-8"}
ARG LC_ALL=${LC_ALL:-"ru_RU.UTF-8"}
ARG LOCALE_GEN=${LOCALE_GEN:-"ru_RU.UTF-8 UTF-8"}
# Timezone defaults
ARG TZ
ENV TZ=${TZ:-"Europe/Moscow"}
# Local vars
ARG LOCAL_DOWNLOAD_PATH=/opt/onec/binary

# Install packages
RUN apt-get update \
    && apt-get install --no-install-recommends -y curl iputils-ping wget curl iproute2 \
    && mkdir $JENKINS_HOME/.ssh \
    && chmod -R 700 $JENKINS_HOME/.ssh \
    && touch $JENKINS_HOME/.ssh/known_hosts \
    && chmod -R 600 $JENKINS_HOME/.ssh/known_hosts

# Locales
RUN apt-get update && apt-get install -y locales \
    && echo "LC_ALL=${LC_ALL}" >> /etc/environment \
    && echo "LANG=${LANG}" > /etc/locale.conf \
    && echo "${LOCALE_GEN}" >> /etc/locale.gen \
    && locale-gen

# Install OPM
RUN wget ${OPM_URL} -O ./onescript.deb \
    && apt-get install -y ./onescript.deb \
    && opm update opm

# Install 1C Platform
RUN mkdir -p ${LOCAL_DOWNLOAD_PATH}
## copy 1c platform binary to image
COPY ./*.run ${LOCAL_DOWNLOAD_PATH}

# Install 1C platforms
ARG CMPS=server,ws,server_admin,client_full,client_thin,client_thin_fib,config_storage_server,liberica_jre
RUN for file in $LOCAL_DOWNLOAD_PATH/*.run; do \
    if [ -f "$file" ]; then \
    chmod +x "$file"; \
    "$file" --mode unattended --enable-components ${CMPS}; \
    fi; \
    done

# Clear distribs
RUN rm -rf ${LOCAL_DOWNLOAD_PATH}/*

# Clear apt cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Vanessa ADD, Vanessa-runner
RUN opm install gitsync \
    && opm install add \
    && opm install vanessa-runner \
    && opm install vanessa-automation \
    && opm install ibcmdrunner