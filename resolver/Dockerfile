FROM ubuntu:22.04

RUN apt-get update -y && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    bind9 bind9-utils bind9-dnsutils dnsutils iputils-ping \
    net-tools vim curl && \
    rm -rf /var/lib/apt/lists/*

# Keep the container alive — BIND9 will be started manually so we can
# edit configs without rebuilding the image
CMD ["tail", "-f", "/dev/null"]
