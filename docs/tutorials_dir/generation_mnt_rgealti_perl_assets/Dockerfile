FROM ubuntu:20.04

SHELL ["/bin/bash", "-c"]

COPY ./dpkg "/tmp/dpkg"

WORKDIR /tmp/dpkg

RUN export DEBIAN_FRONTEND=noninteractive && apt update && apt upgrade -y \
    && apt install -y ./*

ENV ROK4_TMS_DIRECTORY "/usr/share/rok4/tilematrixsets"

WORKDIR /tmp/rok4

CMD ["/bin/bash"]