---
layout: post
title: "Ethernet packets transmission with Ruby"
date:  2022-11-14 00:30:00
tags:  ethernet packets networking
---

Computers use network protocols to transfer data and the basic unit for this is called network packet and
I will use ruby and containers to demonstrate how the packet transmission works.

## What is a network packet and how does it work?

An [ethernet packet or frame](https://en.wikipedia.org/wiki/Ethernet_frame#Header) is part of a complete network 
message and carries address information that helps identify its source and destination. And
we are going to analyze the ethernet packets and communicate 2 hosts using the MAC addresses only.

An ethernet packet has three parts: the header, protocol data unit (PDU) and footer (trailer).

![ethernet encapsulation](/img/network_protocols/ethernet-encapsulation.png)

* The **ethernet header** includes the source and destination mac address and the protocol type.
* The **ethernet payload** contains the data using the underlying structure for other protocols (IP, ICMP, ARP, etc)
* The **footer** (trailer) is used to define the frame check sequence and this is a four-octet cyclic redundancy check.

## MAC Addresses
A media access control address ([MAC address](https://en.wikipedia.org/wiki/MAC_address)) is a unique identifier 
assigned to a network interface controller (NIC) for use as a network address in communications in network.


## Ethernet packet transmission 
To send a packet from the Host A to the Host B without getting too much information about how the operating system 
manages the network interfaces to send the packets we are going to use the [PCAP library](https://www.tcpdump.org/manpages/pcap.3pcap.html) 
from the [Ruby programming language](https://www.ruby-lang.org/en/) through the [pcaprub gem](https://github.com/pcaprub/pcaprub).

![Ethernet packet transmission](/img/network_protocols/ethernet-packet-transmission.png)

The following piece of code will perform the following operations and that does not require root privileges to run:

1. Identifies the default network interface.
2. Builds the byte string that represent an ethernet packet (source and destination mac addresses, type and payload).
3. Injects the ethernet packet to network interface each two seconds forever.

{% highlight ruby linenos %}
require 'pcaprub'

ifname = Pcap.lookupdev

stream = Pcap.open_live(ifname, 0xffff, false, 1)
eth_saddr = '02:42:ab:aa:bb:02'.split(/[:\x2d\x2e\x5f-]+/).collect {|x| x.to_i(16)}.pack('C6')
eth_daddr = '02:42:ab:aa:bb:01'.split(/[:\x2d\x2e\x5f-]+/).collect {|x| x.to_i(16)}.pack('C6')
eth_proto = [0x0800].pack('n')

count = 0
loop do
  payload = "Simple payload #{count += 1}"
  pkt = [eth_daddr, eth_saddr, eth_proto, payload].join.force_encoding('ASCII-8BIT')
  stream.inject(pkt)
  puts "Sent Packet #{count} (#{pkt.size})"
  sleep(2)
end
{% endhighlight %}

## Sending an ethernet packet 
To send an ethernet packet from one host to another, I'll implement a packet sender and an sniffer based on the great 
[packetfu gem](https://github.com/packetfu/packetfu) which is [domain specific language](https://en.wikipedia.org/wiki/Domain-specific_language) 
for packet manipulation, designed for reading and writing packets to an interface or to a libpcap-formatted file.

Each script will be deployed in its own [docker](https://en.wikipedia.org/wiki/Docker_(software)) container running
in the same ethernet network.

## Packet sender
This program is designed to send ethernet packets each 2 seconds from the Host A (02:42:ab:aa:bb:02) to the 
Host B (02:42:ab:aa:bb:01) using the MAC addresses only.

{% highlight ruby linenos %}
require 'packetfu'

eth_pkt = PacketFu::EthPacket.new
eth_pkt.eth_saddr = '02:42:ab:aa:bb:02'
eth_pkt.eth_daddr = '02:42:ab:aa:bb:01'

count = 0
loop do
  eth_pkt.payload = "Ethernet payload #{count += 1}"
  puts '=' * 80
  puts eth_pkt.inspect
  eth_pkt.to_w('eth0')
  sleep(2)
end

{% endhighlight %}

## Packet sniffer
The packet sniffer will capture the traffic in the network interface (eth0) and will print only the
ethernet packets received from the Host A (02:42:ab:aa:bb:02). 

{% highlight ruby linenos %}
#!/usr/bin/env ruby
require 'packetfu'

puts "Capturing network traffic..."
cap = PacketFu::Capture.new(iface: 'eth0', start: true)
cap.stream.each do |raw_packet|
  pkt = PacketFu::Packet.parse(raw_packet)
  next if pkt.is_tcp? || pkt.is_udp? || pkt.is_icmp? || pkt.is_arp?
  next unless pkt.is_eth? && pkt.eth_header.eth_saddr == '02:42:ab:aa:bb:02'
  puts '=' * 80
  puts pkt.inspect
end
{% endhighlight %}

## Docker Compose configuration
The [docker compose](https://docs.docker.com/engine/reference/commandline/compose/) configuration will be used to run 
multiple containers in a [macvlan network](https://docs.docker.com/network/macvlan/) within the l3 mode, which is 
required to allow send raw ethernet packets between containers.

{% highlight yaml linenos %}
version: '3'
services:
  sniffer:
    build: ./sniffer
    container_name: sniffer
    restart: unless-stopped
    mac_address: 02:42:ab:aa:bb:01
    networks:
      - demo-network

  packet_sender:
    build: ./packet_sender
    container_name: packet_sender
    restart: unless-stopped
    mac_address: 2:42:ab:aa:bb:02
    networks:
      - demo-network

networks:
  demo-network:
    driver: macvlan
    driver_opts:
      ipvlan_mode: l3
    ipam:
    driver: default
    config:
      - subnet: 192.168.16.0/24
        gateway: 192.168.16.1

{% endhighlight %}

## Demo
In the following video you can watch how to run the packet sender and the sniffer in two containers.

{% include youtube_embed.html id="3TXfSPK2mnI" %}

### Steps to run the demo
1. Download the source code to run the packet sender and the sniffer from [this file](/src/network_protocols/ethernet_packets.zip)  .
2. Uncompress the file.
3. Execute the following docker commands:

{% highlight bash %}
docker compose -f docker-compose.yml up -d
docker ps
docker logs -f <container_id>
{% endhighlight  %}

NOTE: You will require [docker installed in your machine](https://docs.docker.com/get-docker/).

## Docker commands
* How to build and run the containers

{% highlight bash %}
docker compose -f docker-compose.yml up -d 
{% endhighlight  %}

* How to stop all the containers
{% highlight bash %}
docker compose -f docker-compose.yml stop
{% endhighlight  %}

* How to list the running containers
{% highlight bash %}
docker ps
{% endhighlight  %}

* How watch the logs from the standard output in the containers
{% highlight bash %}
docker logs -f <container_id>
{% endhighlight  %}

* How to stop an specific container
{% highlight bash %}
docker kill <container_id>
{% endhighlight  %}

* How to list the docker images
{% highlight bash %}
docker image ls
{% endhighlight  %}

* How to remove the docker images
{% highlight bash %}
docker rmi <image_id>
{% endhighlight  %}

# Conclusion

Ruby and docker containers are great tools to simulate network communications and to perform packet manipulation.
I hope you enjoyed this article and have great day.

Cheers
## References
* [Ruby](https://www.ruby-lang.org/en/)
* [PcapRub](https://github.com/pcaprub/pcaprub)
* [PacketFu](https://github.com/packetfu/packetfu)
* [Docker](https://docs.docker.com/get-started/)