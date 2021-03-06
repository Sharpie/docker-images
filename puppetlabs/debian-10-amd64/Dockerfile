FROM amd64/debian:10-slim


# === Install Basic Utilities ===

RUN (DEBIAN_FRONTEND=noninteractive; \
     apt-get update; \
     apt-get install -y --no-install-recommends \
       apt-transport-https \
       ca-certificates \
       curl \
       iproute2 \
       net-tools \
       rsync \
       vim-tiny \
       wget; \
     rm -rf /var/lib/apt/lists/*)


# === Install SystemD ===

# NOTE: Could use a smaller init system, like tini, but by rigging up
#       systemd, we can use this image in Beaker tests that manipulate
#       services.
RUN (DEBIAN_FRONTEND=noninteractive && \
     apt-get update && \
     apt-get install -y --no-install-recommends systemd systemd-sysv && \
     rm -rf /var/lib/apt/lists/*)

# Remove units that are not strictly required to operate a container.
# Improves systemd boot time and reduces errors from starting services
# that are managed by the host OS.
#
# Cribbed from the CentOS 7 image:
#   https://hub.docker.com/r/centos/systemd/dockerfile
RUN find /lib/systemd/system/sysinit.target.wants/ \
      -not -type d \
      -not -name systemd-tmpfiles-setup.service \
      -delete; \
    rm -f /lib/systemd/system/multi-user.target.wants/*; \
    rm -f /etc/systemd/system/*.wants/*; \
    rm -f /lib/systemd/system/local-fs.target.wants/*; \
    rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
    rm -f /lib/systemd/system/basic.target.wants/*; \
    rm -f /etc/machine-id


# === Install RSyslog ===

RUN (DEBIAN_FRONTEND=noninteractive; \
     apt-get update && \
     apt-get install -y --no-install-recommends \
       rsyslog; \
     rm -rf /var/lib/apt/lists/*)

# Populate /var/log/syslog from journald
RUN systemctl enable rsyslog.service


# === Install OpenSSH ===

RUN (DEBIAN_FRONTEND=noninteractive && \
     apt-get update && \
     apt-get install -y --no-install-recommends openssh-server openssh-client && \
     rm -rf /var/lib/apt/lists/*)

RUN sed -ri'' -e 's/^#?PermitRootLogin .*/PermitRootLogin yes/' \
      -e 's/^#?PasswordAuthentication .*/PasswordAuthentication yes/' \
      -e 's/^#?UseDNS .*/UseDNS no/' \
      /etc/ssh/sshd_config

# FIXME: Make key configurable
RUN mkdir -p /root/.ssh && \
    curl -G https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub \
      -o /root/.ssh/authorized_keys

# FIXME: Debian 10 uses socket activation for SSH. We mask that in order
#        to run the "traditional" service since ServerSpec "port listening"
#        tests don't work that well with Docker containers at the moment.
RUN systemctl mask ssh.socket && systemctl enable ssh.service

# Java installation fails if man1 does not exist
#   https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=863199
RUN mkdir -p /usr/share/man/man1


# === Runtime Configuration ===
# NOTE: Compacting a debian-slim image via COPY seems to bloat the final
#       image by ~75 MB instead of reducing it.

# NOTE: Technically, should also have `--tmpfs /tmp:exec`. But, tools
#       that use the Docker API to upload files, such as Bolt, can't write
#       content to a mount.
#       https://developers.redhat.com/blog/2016/09/13/running-systemd-in-a-non-privileged-container/
LABEL docker-run-args="--tmpfs=/run --tmpfs /run/lock --volume=/sys/fs/cgroup:/sys/fs/cgroup:ro"
LABEL org.opencontainers.image.source="https://github.com/Sharpie/puppet-dev-images"
ENV container docker
STOPSIGNAL SIGRTMIN+3
VOLUME [ "/sys/fs/cgroup", "/tmp", "/run" ]
EXPOSE 22

CMD [ "/sbin/init" ]
