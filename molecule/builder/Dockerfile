# April 28th, 2018 ubuntu:trusty
FROM ubuntu@sha256:b8855dc848e2622653ab557d1ce2f4c34218a9380cceaa51ced85c5f3c8eb201

# additional meta-data makes it easier to clean up, find
LABEL org="Freedom of the Press"
LABEL image_name="trusty-sd-builder-app"

RUN apt-get -y update && apt-get upgrade -y && apt-get install -y \
        aptitude \
        devscripts \
        git \
        gdb \
        gnupg2 \
        haveged \
        libffi-dev \
        libssl-dev \
        make \
        ntp \
        inotify-tools \
        paxctl \
        python \
        python-dev \
        python-pip \
        rsync \
        ruby \
        secure-delete \
        sqlite \
        unzip \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*

RUN paxctl -cm /usr/bin/python2.7 && mkdir -p /tmp/build
