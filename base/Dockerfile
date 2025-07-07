# Builder stage: compile WireGuard Go and tools
FROM ubuntu:24.04 AS ubuntu-builder

ARG wg_go_tag=0.0.20250522
ARG wg_tools_tag=v1.0.20250521

RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      ca-certificates \
      git \
      make \
      golang-go \
      build-essential && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Clone, build and install wireguard-go
RUN git clone https://git.zx2c4.com/wireguard-go && \
    cd wireguard-go && \
    git checkout $wg_go_tag && \
    make && make install

ENV WITH_WGQUICK=yes

# Clone, build and install wireguard-tools
RUN git clone https://git.zx2c4.com/wireguard-tools && \
    cd wireguard-tools && \
    git checkout $wg_tools_tag && \
    cd src && \
    make && make install

# VNC stage: desktop, VNC, Docker, Chromium, WireGuard, etc.
FROM ubuntu:24.04 AS ubuntu-vnc

ENV DEBIAN_FRONTEND=noninteractive
ENV SHELL=/bin/bash

# 1) Install core packages, XFCE, VNC, Docker, Supervisor, etc., plus Chromium dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      software-properties-common sudo \
      xfce4-goodies xfce4 tigervnc-standalone-server \
      docker.io supervisor jq curl \
      zsh vim netcat-traditional dbus-x11 \
      python3 python3-pip python3-venv golang-go \
      linux-generic xserver-xorg-legacy && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# 2) Add XTradeb PPA & install Chromium (deb, no snap)
RUN add-apt-repository -y ppa:xtradeb/apps && \
    apt-get update && \
    apt-get install -y --no-install-recommends chromium && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# 3) Configure ubuntu user: sudo, docker group, default shell
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/90-sudo && \
    usermod -aG sudo,docker ubuntu && \
    chsh -s /bin/bash ubuntu

# 4) Prepare Xwrapper to allow any user to start X
RUN mkdir -p /etc/X11 && \
    printf 'allowed_users=anybody\nneeds_root_rights=yes\n' > /etc/X11/Xwrapper.config

# 5) Remove unused browser launchers
RUN rm -f \
      /usr/share/applications/firefox.desktop \
      /usr/share/applications/web-browser.desktop \
      /usr/share/applications/epiphany.desktop \
      /usr/share/applications/org.gnome.Epiphany.desktop

# 6) Wrap Chromium to disable sandbox, register as default browser, configure xdg
RUN mv /usr/bin/chromium /usr/bin/chromium.real && \
    printf '#!/bin/sh\nexec /usr/bin/chromium.real --no-sandbox "$@"\n' > /usr/bin/chromium && \
    chmod +x /usr/bin/chromium && \
    update-alternatives --install /usr/bin/x-www-browser x-www-browser /usr/bin/chromium 200 && \
    update-alternatives --install /usr/bin/gnome-www-browser gnome-www-browser /usr/bin/chromium 200 && \
    update-alternatives --set x-www-browser /usr/bin/chromium && \
    update-alternatives --set gnome-www-browser /usr/bin/chromium && \
    mkdir -p /etc/xdg/xdg-xfce && \
    printf '[Default Applications]\nx-scheme-handler/http=chromium.desktop\nx-scheme-handler/https=chromium.desktop\n' \
      > /etc/xdg/xdg-xfce/mimeapps.list

# 7) Copy WireGuard binaries from builder
COPY --from=ubuntu-builder /usr/bin/wireguard-go /usr/bin/wg* /usr/bin/

# 8) Setup scripts, supervisor and VNC entrypoint
RUN mkdir -p /scripts /etc/wireguard

COPY supervisor.conf.d/ /etc/supervisor/conf.d/
COPY launcher.sh /scripts/launcher.sh

# Ensure ICEauthority exists and is owned by ubuntu
RUN touch /root/.ICEauthority && chown ubuntu:ubuntu /root/.ICEauthority

# 9) Tweak XFCE panel (remove CPU graph & power manager icons)
RUN sed -i 's/.*"cpugraph".*//g' /etc/xdg/xfce4/panel/default.xml && \
    sed -i 's/.*"power-manager-plugin".*//g' /etc/xdg/xfce4/panel/default.xml

WORKDIR /home/ubuntu

HEALTHCHECK --interval=30s --timeout=5s CMD nc -vz 127.0.0.1 5900

ENTRYPOINT ["bash", "/scripts/launcher.sh"]

