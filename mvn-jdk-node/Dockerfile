ARG mvn_version=3.6.3
ARG jdk_version=8
ARG node_version=15.4.0
FROM maven:${mvn_version}-jdk-${jdk_version}

ENV NODE_VERSION=${node_version:-15.4.0}

RUN set -ex \
    && cat /etc/resolv.conf \
    && ls -l /etc/resolv.conf \
    && iptables --list \
    && route \
    && /sbin/ifconfig \
    && ip addr show eth0 \
    && ping -n 2 deb.debian.org
