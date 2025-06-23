# Defaults
ARG JENKINS_SSH_DOCKERHUB_IMAGE_NAME=${JENKINS_SSH_DOCKERHUB_IMAGE_NAME:-"jenkins/ssh-agent"}
ARG JENKINS_SSH_DOCKERHUB_IMAGE_TAG=${JENKINS_SSH_DOCKERHUB_IMAGE_TAG:-"latest-jdk21"}

FROM ${JENKINS_SSH_DOCKERHUB_IMAGE_NAME}:${JENKINS_SSH_DOCKERHUB_IMAGE_TAG} AS ssh-agent

ARG DEBIAN_FRONTEND=noninteractive

# JENKINS_AGENT_HOME - prebuild in base image

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
# VNC
ARG VNC_DISPLAY
ARG VNC_RESOLUTION
ARG VNC_USER
ARG VNC_DISPLAY=${VNC_DISPLAY:-":1"}
ARG VNC_RESOLUTION=${VNC_RESOLUTION:-"1280x720"}
ARG VNC_USER=${VNC_USER:-"jenkins"}
## use Docker-secret for passwords, not send sensitive data by Dockerfile

# Run updates
RUN pwconv
RUN pam-auth-update

# Install packages
RUN apt-get update \
    && apt-get install --no-install-recommends -y curl sudo iputils-ping wget iproute2 unzip \
    # --> Certificates
    ca-certificates ssl-cert \
    # <-- Certificates
    # --> Install packages for VNC
    xfce4-terminal mate-desktop-environment-extras aqemu aria2 \
    qemu-system-x86 htop chromium screen tightvncpasswd tigervnc-standalone-server python3-pip \
    python3-websockify python3 git fuse libfuse2 xdotool \
    # <-- Install packages for VNC
    # --> For 1C
    libwebkit2gtk-4.0-37 \
    # <-- For 1C
    # --> Locales
    locales \
    # <-- Locales
    && mkdir -p "${JENKINS_AGENT_HOME}/.ssh" \
    && chmod -R 700 "${JENKINS_AGENT_HOME}/.ssh" \
    && touch "${JENKINS_AGENT_HOME}/.ssh/known_hosts" \
    && chmod -R 600 "${JENKINS_AGENT_HOME}/.ssh/known_hosts" \
    # --> Clear apt cache
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Remove packages
RUN (apt-cache show lxlock && apt-get remove -y lxlock) || true \
    && (apt-cache show light-locker && apt-get remove -y light-locker) || true \
    && (apt-cache show xscreensaver-data && apt-get remove -y xscreensaver-data) || true \
    && (apt-cache show xscreensaver && apt-get remove -y xscreensaver) || true

# Prepare configs for VNC
RUN sed -i '/@xscreensaver -no-splash/d' /etc/xdg/lxsession/LXDE/autostart || echo "Конфигурация не существует, игнорирую"
RUN git clone https://github.com/novnc/noVNC.git noVNC
RUN mkdir -p "${JENKINS_AGENT_HOME}/.vnc" "/tmp"
RUN chmod -R 777 "${JENKINS_AGENT_HOME}/.vnc" "/tmp"

RUN touch "${JENKINS_AGENT_HOME}/.Xauthority" && chown ${VNC_USER}:${VNC_USER} "${JENKINS_AGENT_HOME}/.Xauthority"
ENV HOME=${JENKINS_AGENT_HOME} \
    PATH=${JENKINS_AGENT_HOME}/.local/bin:$PATH

# Locales
RUN echo "LC_ALL=${LC_ALL}" >> /etc/environment \
    && echo "LANG=${LANG}" > /etc/locale.conf \
    && echo "${LOCALE_GEN}" >> /etc/locale.gen \
    && locale-gen

# Install OPM (with recomends, cause several packages could be changed)
RUN wget --progress=dot:giga "${OPM_URL}" -O ./onescript.deb \
    && apt-get update && apt-get install -y libmono-i18n4.0-all libmono-system-core4.0-cil libmono-system4.0-cil mono-runtime \
    && apt-get install --no-install-recommends -y ./onescript.deb \
    # --> Clear apt cache
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && opm update opm

# Install 1C Platform
RUN mkdir -p "${LOCAL_DOWNLOAD_PATH}"
## copy 1c platform binary to image
COPY ./*.run "${LOCAL_DOWNLOAD_PATH}"

# Install 1C platforms
ARG CMPS=server,ws,server_admin,client_full,client_thin,client_thin_fib,config_storage_server,liberica_jre
RUN for file in "$LOCAL_DOWNLOAD_PATH"/*.run; do \
    if [ -f "$file" ]; then \
    chmod +x "$file"; \
    # --- >
    "$file" --mode unattended --enable-components ${CMPS}; \
    fi; \
    done

# Clear distribs (check for empty variable to avoid root folder)
RUN rm -rf "${LOCAL_DOWNLOAD_PATH:?}"/*

# Install SonarQube scanner
ARG SONAR_SCANNER_VERSION
ARG SONAR_SCANNER_VERSION=${SONAR_SCANNER_VERSION:-"7.0.2.4839"}
RUN export SONAR_SCANNER_VERSION="${SONAR_SCANNER_VERSION}" \
    && export SONAR_SCANNER_HOME="${JENKINS_AGENT_HOME}/.sonar/sonar-scanner-${SONAR_SCANNER_VERSION}-linux-x64" \
    && export PATH="${SONAR_SCANNER_HOME}/bin:$PATH"

RUN mkdir -p "${JENKINS_AGENT_HOME}/.sonar/" \
    && wget --progress=dot:giga "https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${SONAR_SCANNER_VERSION}-linux-x64.zip" --output-document="${JENKINS_AGENT_HOME}/.sonar/sonar-scanner.zip" \
    && unzip -o "${JENKINS_AGENT_HOME}/.sonar/sonar-scanner.zip" -d "${JENKINS_AGENT_HOME}/.sonar/" \
    && chown -R jenkins:jenkins "/home/jenkins/.sonar"

# Export to environment
ENV SONAR_SCANNER_VERSION=${SONAR_SCANNER_VERSION}
ENV SONAR_SCANNER_HOME="${JENKINS_AGENT_HOME}/.sonar/sonar-scanner-${SONAR_SCANNER_VERSION}-linux-x64"
ENV PATH=${SONAR_SCANNER_HOME}/bin:${PATH}
# Add PATH for all users
## For bash users / sh users
RUN echo "export PATH=\"${SONAR_SCANNER_HOME}/bin:${PATH}\"" >> /etc/bash.bashrc \
    && echo "export PATH=\"${SONAR_SCANNER_HOME}/bin:${PATH}\"" >> /etc/profile

# --->
# Install Vanessa ADD, Vanessa-runner
RUN opm install gitsync \
    && opm install add \
    && opm install vanessa-runner \
    && opm install vanessa-automation \
    && opm install ibcmdrunner


# EXPOSE ports
# SSH
EXPOSE 22
# NoVNC-proxy. HTTP access to GUI
EXPOSE 7860

# Forward ARG values to ENV to use in CMD
ENV DISPLAY=${VNC_DISPLAY}
ENV VNC_USER=${VNC_USER}
ENV VNC_RESOLUTION=${VNC_RESOLUTION}

# Get password from secrets and change users' password to it
RUN --mount=type=secret,id=VNC_PASSWORD,env=VNC_PASSWORD,mode=0444,required=true \
    VNC_PASSWORD=${VNC_PASSWORD:-"123456"}; \
    echo "${VNC_USER}:${VNC_PASSWORD}" | chpasswd; \
    echo "${VNC_PASSWORD}" | vncpasswd -f > ${JENKINS_AGENT_HOME}/.vnc/passwd

# Add SSH keys for external GIT repositories 
RUN ssh-keyscan -t ed25519 github.com >> "${JENKINS_AGENT_HOME}/.ssh/known_hosts" \
    && ssh-keyscan -t ed25519 gitlab.com >> "${JENKINS_AGENT_HOME}/.ssh/known_hosts"

# ------------------------------------------------------------------------------
# https://docs.docker.com/reference/build-checks/json-args-recommended/
# Note currently workdir folder is: WORKDIR /home/jenkins
COPY --chmod=755 <<EOT /entrypoint.sh
#!/usr/bin/env bash
set -e;
## --> Background
vncserver -SecurityTypes VncAuth -rfbauth "${JENKINS_AGENT_HOME}/.vnc/passwd" -display "${DISPLAY}" -geometry "${VNC_RESOLUTION}" 2>&1 &
"${JENKINS_AGENT_HOME}/noVNC/utils/novnc_proxy" --vnc 0.0.0.0:5901 --listen 0.0.0.0:7860 2>&1 &
## <-- Background
setup-sshd
EOT
SHELL ["/bin/bash", "-c"]
ENTRYPOINT ["/entrypoint.sh"]
CMD [ "/usr/local/bin/setup-sshd" ]