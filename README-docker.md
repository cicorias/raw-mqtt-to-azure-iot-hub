from: https://stackoverflow.com/a/69055795/140618

To try it out run docker exec -it ipsec-playground_bob_1 tcpdump and 
  docker exec -it ipsec-playground_alice_1 ping 172.29.0.5. You should see pings from alice reaching bob.

ubuntu-with-tools is a simple ubuntu image with some things that I wanted installed, here's the Dockerfile:

FROM ubuntu
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y iproute2 inetutils-ping curl host mtr-tiny tcpdump iptables \
    && rm -rf /var/lib/apt/lists/*



Unfortunately this use-case isn't really supported by bridge networks (or any other kind of docker network). The problem is that bridge networks use your host machine as the gateway. It does this by creating a virtual interface for your host on the bridge network and doing some host configuration to implement the necessary routing. The gateway option only configures what IP address will be assigned to this virtual interface.

This doesn't seem to be well documented, I think it's considered an implementation detail of docker networking. I found it out from a couple of forum posts: source 1, source 2.

Edit: Here's my working docker-compose file. This has two private networks, each with it's own gateway, ie alice <-> moon <-> sun <-> bob. The magic is in the ip and iptables commands inside the command: block run by each container. Ignore the tail -f /dev/null, it's just a command that will never finish which means the container stays running until you kill it.

version: "3.3"

services:

  alice:
    image: ubuntu-with-tools
    cap_add:
      - NET_ADMIN
    hostname: alice
    networks:
      moon-internal:
        ipv4_address: 172.28.0.3
    command: >-
      sh -c "ip route del default &&
      ip route add default via 172.28.0.2 &&
      tail -f /dev/null"

  moon:
    image: ubuntu-with-tools
    cap_add:
      - NET_ADMIN
    hostname: moon
    networks:
      moon-internal:
        ipv4_address: 172.28.0.2
      internet:
        ipv4_address: 172.30.0.2
    command: >-
      sh -c "iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE &&
      iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE &&
      iptables -A FORWARD -i eth1 -j ACCEPT &&
      iptables -A FORWARD -i eth0 -j ACCEPT &&
      ip route add 172.29.0.0/16 via 172.30.0.4 &&
      tail -f /dev/null"


  sun:
    image: ubuntu-with-tools
    cap_add:
      - NET_ADMIN
    hostname: sun
    networks:
      sun-internal:
        ipv4_address: 172.29.0.4
      internet:
        ipv4_address: 172.30.0.4
    command: >-
      sh -c "iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE &&
      iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE &&
      iptables -A FORWARD -i eth1 -j ACCEPT &&
      iptables -A FORWARD -i eth0 -j ACCEPT &&
      ip route add 172.28.0.0/16 via 172.30.0.2 &&
      tail -f /dev/null"

  bob:
    image: ubuntu-with-tools
    cap_add:
      - NET_ADMIN
    hostname: bob
    networks:
      sun-internal:
        ipv4_address: 172.29.0.5
    command: >-
      sh -c "ip route del default &&
      ip route add default via 172.29.0.4 &&
      tail -f /dev/null"


networks:
  moon-internal:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
  sun-internal:
    driver: bridge
    ipam:
      config:
        - subnet: 172.29.0.0/16
  internet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16
