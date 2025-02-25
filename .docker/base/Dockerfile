#
# STAGE 1: SERVER
#
FROM golang:1.18-bullseye as server
WORKDIR /src

#
# install dependencies
RUN set -eux; apt-get update; \
    apt-get install -y --no-install-recommends git cmake make libx11-dev libxrandr-dev libxtst-dev \
    libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly; \
    #
    # install libclipboard
    set -eux; \
    cd /tmp; \
    git clone --depth=1 https://github.com/jtanx/libclipboard; \
    cd libclipboard; \
    cmake .; \
    make -j4; \
    make install; \
    rm -rf /tmp/libclipboard; \
    #
    # clean up
    apt-get clean -y; \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/*

#
# build server
COPY server/ .
RUN go get -v -t -d . && go build -o bin/neko cmd/neko/main.go

#
# STAGE 2: CLIENT
#
FROM node:14-bullseye-slim as client
WORKDIR /src

#
# install dependencies
COPY client/package*.json ./
RUN npm install

#
# build client
COPY client/ .
RUN npm run build

#
# STAGE 3: RUNTIME
#
FROM debian:bullseye-slim

#
# avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

#
# set custom user
ARG USERNAME=neko
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN set -eux; \
    #
    # add non-free repo for intel drivers
    echo deb http://deb.debian.org/debian bullseye main contrib non-free > /etc/apt/sources.list; \
    echo deb http://deb.debian.org/debian-security/ bullseye-security main contrib non-free >> /etc/apt/sources.list; \
    echo deb http://deb.debian.org/debian bullseye-updates main contrib non-free >> /etc/apt/sources.list; \
    apt-get update; \
    #
    # install dependencies
    apt-get install -y --no-install-recommends wget ca-certificates supervisor; \
    apt-get install -y --no-install-recommends pulseaudio dbus-x11 xserver-xorg-video-dummy; \
    apt-get install -y --no-install-recommends libcairo2 libxcb1 libxrandr2 libxv1 libopus0 libvpx6; \
    #
    # intel driver + vaapi
    apt-get install -y --no-install-recommends intel-media-va-driver-non-free libva2 vainfo; \
    #
    # gst + vaapi plugin
    apt-get install -y --no-install-recommends libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
                    gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-pulseaudio \
                    gstreamer1.0-vaapi ;\
    #
    # fonts
    apt-get install -y --no-install-recommends fonts-takao-mincho; \
    #
    # create a non-root user
    groupadd --gid $USER_GID $USERNAME; \
    useradd --uid $USER_UID --gid $USERNAME --shell /bin/bash --create-home $USERNAME; \
    adduser $USERNAME audio; \
    adduser $USERNAME video; \
    adduser $USERNAME pulse; \
    #
    # setup pulseaudio
    mkdir -p /home/$USERNAME/.config/pulse/; \
    echo "default-server=unix:/tmp/pulseaudio.socket" > /home/$USERNAME/.config/pulse/client.conf; \
    #
    # workaround for an X11 problem: http://blog.tigerteufel.de/?p=476
    mkdir /tmp/.X11-unix; \
    chmod 1777 /tmp/.X11-unix; \
    chown $USERNAME /tmp/.X11-unix/; \
    #
    # make directories for neko
    mkdir -p /etc/neko /var/www /var/log/neko; \
    chmod 1777 /var/log/neko; \
    chown $USERNAME /var/log/neko/; \
    chown -R $USERNAME:$USERNAME /home/$USERNAME; \
    #
    # clean up
    apt-get clean -y; \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/*

#
# copy config files
COPY .docker/base/dbus /usr/bin/dbus
COPY .docker/base/default.pa /etc/pulse/default.pa
COPY .docker/base/supervisord.conf /etc/neko/supervisord.conf
COPY .docker/base/xorg.conf /etc/neko/xorg.conf
COPY .docker/base/add-render-group.sh /usr/bin/add-render-group.sh

#
# set default envs
ENV USER=$USERNAME
ENV DISPLAY=:99.0
ENV NEKO_PASSWORD=neko
ENV NEKO_PASSWORD_ADMIN=admin
ENV NEKO_BIND=:8080
ENV RENDER_GID=

#
# copy static files from previous stages
COPY --from=server /src/bin/neko /usr/bin/neko
COPY --from=client /src/dist/ /var/www

HEALTHCHECK --interval=10s --timeout=5s --retries=8 \
    CMD wget -O - http://localhost:${NEKO_BIND#*:}/health || exit 1

#
# run neko
CMD ["/usr/bin/supervisord", "-c", "/etc/neko/supervisord.conf"]
