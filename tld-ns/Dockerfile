FROM ubuntu:22.04

RUN apt-get update -y && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    bind9 bind9-utils bind9-dnsutils dnsutils iputils-ping \
    net-tools vim curl && \
    rm -rf /var/lib/apt/lists/*

COPY etc/named.conf /etc/bind/named.conf
COPY zones/ /etc/bind/zones/

CMD ["named", "-g", "-c", "/etc/bind/named.conf"]
