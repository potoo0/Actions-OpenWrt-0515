# set proxy for build:
# 1. set host porxy:
#       HTTP_PROXY="http://127.0.0.1:1087"
#       HTTPS_PROXY="http://127.0.0.1:1087"
# 2. docker build --network host .
FROM ubuntu:jammy

ENV TZ=Asia/Shanghai LANG=C.UTF-8 TERM=xterm-256color SSH_SERVER=true

RUN DEBIAN_FRONTEND=noninteractive \
    && deps='ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential bzip2 ccache cmake cpio curl device-tree-compiler fastjar flex gawk gettext gcc-multilib g++-multilib git gperf haveged help2man intltool libc6-dev-i386 libelf-dev libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses5-dev libncursesw5-dev libreadline-dev libssl-dev libtool lrzsz mkisofs msmtp nano ninja-build p7zip p7zip-full patch pkgconf python2.7 python3 qemu-utils rsync scons squashfs-tools subversion swig texinfo uglifyjs upx-ucl unzip vim wget xmlto xxd zlib1g-dev' \
    && apt-get update -qq \
    && apt-get upgrade -qqy \
    && apt-get install -qqy $deps \
    && apt-get autoremove --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
#    && mkdir -p /var/run/sshd \
#    && rm -f /etc/ssh/ssh_host_*key*

EXPOSE 22
