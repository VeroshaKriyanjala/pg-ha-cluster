# Minimal Ubuntu 24.04 image that runs systemd as PID 1, so the cluster's
# services (etcd, patroni, haproxy) can be managed exactly like on a real VM.
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
ENV container=docker

# NOTE: we deliberately KEEP /var/lib/apt/lists. The playbook's apt-update step
# uses cache_valid_time, so on a freshly built image it would consider the cache
# fresh, skip the update, and then fail to find packages if the lists were gone.
RUN apt-get update && apt-get install -y --no-install-recommends \
        systemd systemd-sysv \
        python3 python3-apt python3-venv python3-pip \
        sudo ca-certificates curl gnupg \
        iproute2 iputils-ping procps psmisc

# Drop units that are meaningless (or harmful) inside a container.
RUN find /lib/systemd/system /etc/systemd/system -path '*.wants/*' \
      \( -name '*udev*' -o -name '*getty*' -o -name '*networkd*' \
         -o -name '*resolved*' -o -name '*timesyncd*' \) \
      -exec rm -f {} + 2>/dev/null || true

# systemd shuts down cleanly on SIGRTMIN+3
STOPSIGNAL SIGRTMIN+3

CMD ["/sbin/init"]
