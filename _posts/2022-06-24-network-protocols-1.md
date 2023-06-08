---
layout: post
title:  "Network protocols with Ruby - 1"
date:   2022-06-24 16:30:00
tags: ruby rust networking security scanning
---

I'm a fan of network communications and computer security, and I will publish a set of
articles to understand basic networking concepts that could be useful to attack network protocols.

This is a personal project with non-profit purposes, just to have fun and learn about Networking, Security, Ruby.
I hope you enjoy reading this and learn with me.

I will use the [Ruby programming language](https://www.ruby-lang.org/en/) and the [bundler](https://bundler.io/) tool to install examples dependencies.

Let's start defining basic concepts.

# What is a network?

A network is a set of 2 or more device connected together sharing information. Example:

![Basic Network](/img/network_protocols/basic_network.png){:height="500px" width="600px"}

We can see four devices with different operating systems connected to the same network, where they communicate and share data using common network protocols.

# What is a network protocol?

A [network protocol](https://en.wikipedia.org/wiki/Communication_protocol) is defined by a set of rules that help to establish communication between multiple hardware 
devices running different operating systems with its own communication stack.

A network protocol is used for:

* Keep session state.
* Identifying nodes by addressing.
* Controlling the transmission flow.
* Ensuring the order of the transmitted data.
* Detecting an correcting errors.
* Formatting and encoding data.

# The Internet Protocol Suite (IPS)

The [Internet protocol suite](https://en.wikipedia.org/wiki/Internet_protocol_suite), commonly known as TCP/IP, is the set of communications protocols used in the Internet and similar computer networks. The current foundational protocols in the suite are the [Transmission Control Protocol (TCP)](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) and the Internet Protocol (IP), as well as the [User Datagram Protocol (UDP)](https://en.wikipedia.org/wiki/User_Datagram_Protocol).

Network protocols send network traffic over internet using the following protocol stack with 4 layers:

![IPS](/img/network_protocols/internet_protocol_suite.png)

* Link layer: This is the lowest level and describes the physical mechanism used to transfer information between the devices on a local network.
* Internet layer: This layer provides the mechanisms for addressing network devices. The devices don't need to be on the local network. This level contains the IP which could be IPv4 or IPv6.
* Transport layer: This layer is responsible for connections between clients and servers, sometimes ensuring the correct order of packets and providing a service multiplexing. TCP and UDP protocols operate at this layer.
* Application layer: This layer contains network protocols such as HTTP, DNS, SMTP, MongoDB, SSH, etc.

Each layer interacts only with the layer above and below it, there must be some external interactions in the stack. The link layer interacts with physical network connection, transmitting data in physical medium, such
as pulses of electricity or light and the application layer interacts with the user application.

# Data encapsulation

The IPS is composed of multiple layers, each of which is constructed upon the one above it and is capable of encapsulating data from the layer above to allow for layer-to-layer communication. Each layer's data transmission is referred to as a protocol data unit (PDU).

## Headers, Footers and Addresses
The PDU in each layer contains the payload that is being transmitted. it's a common to prefix a header -- which contains information required for the payload data to be transmitted, such as source and destination address in the network. Sometimes a PDU also has a footer that is suffixed to the payload data and contains values needed to ensure correct transmission, such as error checking information.

![IPS](/img/network_protocols/ips_data_encapsulation.png)

1. The application layer contains the payload specific for the content parser of the application that will read this data, and this can be clear text, binary or encrypted data.

2. The transport layer contains a TCP/UDP header with the source and destination port number. These port numbers allow a single device to have multiple unique network connections and they 
range from 0 to 65535. The TCP payload and header are commonly called a segment, whereas a UDP payload and header are commonly called a datagram.

3. The Internet layer has an IP payload with header that includes a source and a destination address. The destination address allows the data to be sent to a specific device on the network. The source address allows the receiver of the data to know which node sent the data and allows the receiver to reply to the sender. An IP payload and header are commonly called a packet.

4. The Link layer has the ethernet payload with a source and destination addresses. Ethernet uses a 64-bit value called a Media Access Control (MAC) address, which is typically set during manufacture of the Ethernet adapter. The MAC addresses are written as a series of hexadecimal numbers separated by dashes or colons, example: 24:E5:D9:2F:3D:4C. The Ethernet payload, including the header and footer is referred to as a frame.

## Example 

There was too much theory there, so let's examine the data encapsulation layer in a real network communication between two computers by starting with a simple example.

In the following example we will capture and display the network communications between the ftp client and a public anonymous FTP server.

![IPS](/img/network_protocols/ftp_client_server.png)

## Sniffer
To capture and display the network traffic we will require a simple sniffer 
{% highlight ruby linenos %}
require 'packetfu'
include PacketFu

iface = ARGV[0] || PacketFu::Utils.default_int
filter = 'host ftp.netbsd.org and tcp and port 21'
cap = Capture.new(iface: iface, start: true, filter: filter)

puts "Capturing network traffic..."
cap.stream.each do |p|
  pkt = Packet.parse(p)
  next unless pkt.is_ip?

  next if pkt.ip_saddr == Utils.ifconfig(iface)[:ip_saddr]

  packet_info = [pkt.ip_saddr, pkt.ip_daddr, pkt.size, pkt.proto.last]
  puts '=' * 80
  puts "%-15s -> %-15s %-4d %s" % packet_info
  puts pkt.dissect
  puts ""
end
{% endhighlight %}
NOTE: You can comment or remove the line 13 if you want observe all the packets from the client and the server.

## FTP client
To generate the traffic we will use a very simple ftp client.
{% highlight ruby linenos %}
require 'net/ftp'

host = 'ftp.netbsd.org'
ftp = Net::FTP.new(host, debug_mode: true)
ftp.login('anonymous')
ftp.chdir('pub')
ftp.list('*').each do |f|
  puts f
end
ftp.close()
{% endhighlight %}

* [Ruby code](/src/network_protocols/example1.tar.xz)  
* [Sniffer in Rust](/src/network_protocols/example1_rust.tar.xz)
* [FTP client in Rust](/src/network_protocols/example1_ftp_client_rust.tar.xz)

# Demo

{% include youtube_embed.html id="JlXi5Y4IDZ8" %}

# Executed commands
{% highlight bash linenos %}
bundle install
rvmsudo bundle exec ruby sniffer.rb 
bundle exec ruby ftp_client.rb
{% endhighlight %}

# Ouput examples

In the output of the sniffer there can be observed the 4 layer of encapsulation per packet, for example:
## TCP Handshake packet
This packet contains the Link (EthHeader), Internet (IPHeader), and Transport (TCPHeader) layers because it is part of the [TCP protocol handshake](https://developer.mozilla.org/en-US/docs/Glossary/TCP_handshake),
for example the A and S flags can be seen in the TCPHeader and this does not include anything related the application layer, which means the application payload is empty.

{% highlight bash linenos %}
================================================================================
199.233.217.201 -> 192.168.11.46   74   TCP
--EthHeader-------------------------------------------------------------------
  eth_dst      a0:78:17:92:23:52                         PacketFu::EthMac
  eth_src      3c:37:86:fb:96:9d                         PacketFu::EthMac
  eth_proto    0x0800                                    StructFu::Int16
--IPHeader--------------------------------------------------------------------
  ip_v         4                                         Integer
  ip_hl        5                                         Integer
  ip_tos       0                                         StructFu::Int8
  ip_len       60                                        StructFu::Int16
  ip_id        0x0000                                    StructFu::Int16
  ip_frag      16384                                     StructFu::Int16
  ip_ttl       52                                        StructFu::Int8
  ip_proto     6                                         StructFu::Int8
  ip_sum       0xd932                                    StructFu::Int16
  ip_src       199.233.217.201                           PacketFu::Octets
  ip_dst       192.168.11.46                             PacketFu::Octets
--TCPHeader-------------------------------------------------------------------
  tcp_src      21                                        StructFu::Int16
  tcp_dst      50706                                     StructFu::Int16
  tcp_seq      0x0cbde2e8                                StructFu::Int32
  tcp_ack      0x116fb67d                                StructFu::Int32
  tcp_hlen     10                                        PacketFu::TcpHlen
  tcp_reserved 0                                         PacketFu::TcpReserved
  tcp_ecn      0                                         PacketFu::TcpEcn
  tcp_flags    .A..S.                                    PacketFu::TcpFlags
  tcp_win      4096                                      StructFu::Int16
  tcp_sum      0x523e                                    StructFu::Int16
  tcp_urg      0                                         StructFu::Int16
  tcp_opts     MSS:1412,NOP,WS:11,SACKOK,TS:1;2756728392 PacketFu::TcpOptions
{% endhighlight %}

## FTP server response (Packet with application payload)
This packet that includes the Link (EthHeader), Internet (IPHeader), Transport (TCPHeader) and Application layers because it presents the server's initial response via the FTP protocol.In this case, the payload is in clear text, and the ftp client can understand it.
{% highlight bash linenos %}
================================================================================
199.233.217.201 -> 192.168.11.46   127  TCP
--EthHeader-------------------------------------------------
  eth_dst      a0:78:17:92:23:52       PacketFu::EthMac
  eth_src      3c:37:86:fb:96:9d       PacketFu::EthMac
  eth_proto    0x0800                  StructFu::Int16
--IPHeader--------------------------------------------------
  ip_v         4                       Integer
  ip_hl        5                       Integer
  ip_tos       0                       StructFu::Int8
  ip_len       113                     StructFu::Int16
  ip_id        0x437e                  StructFu::Int16
  ip_frag      16384                   StructFu::Int16
  ip_ttl       52                      StructFu::Int8
  ip_proto     6                       StructFu::Int8
  ip_sum       0x957f                  StructFu::Int16
  ip_src       199.233.217.201         PacketFu::Octets
  ip_dst       192.168.11.46           PacketFu::Octets
--TCPHeader-------------------------------------------------
  tcp_src      21                      StructFu::Int16
  tcp_dst      50706                   StructFu::Int16
  tcp_seq      0x0cbde2e9              StructFu::Int32
  tcp_ack      0x116fb67d              StructFu::Int32
  tcp_hlen     8                       PacketFu::TcpHlen
  tcp_reserved 0                       PacketFu::TcpReserved
  tcp_ecn      0                       PacketFu::TcpEcn
  tcp_flags    .AP...                  PacketFu::TcpFlags
  tcp_win      2                       StructFu::Int16
  tcp_sum      0xbb0f                  StructFu::Int16
  tcp_urg      0                       StructFu::Int16
  tcp_opts     NOP,NOP,TS:2;2756728496 PacketFu::TcpOptions
------------------------------------------------------------------
00-01-02-03-04-05-06-07-08-09-0a-0b-0c-0d-0e-0f---0123456789abcdef
------------------------------------------------------------------
32 32 30 20 66 74 70 2e 4e 65 74 42 53 44 2e 6f   220 ftp.NetBSD.o
72 67 20 46 54 50 20 73 65 72 76 65 72 20 28 4e   rg FTP server (N
65 74 42 53 44 2d 66 74 70 64 20 32 30 31 38 30   etBSD-ftpd 20180
34 32 38 29 20 72 65 61 64 79 2e 0d 0a            428) ready...

{% endhighlight %}

You can get the same results with tools like [wireshark](https://www.wireshark.org/), [tshark](https://www.wireshark.org/docs/man-pages/tshark.html) or [tcpdump](https://www.tcpdump.org/) but it is very easy implementing this with Ruby.

Routing, host name resolution (dns), protocol analysis, protocol implementation, and other topics were not covered here and will be covered in subsequent posts.

Happy Friday everyone!
# References
1. [Ruby Version Manager - Installing RVM](https://rvm.io/rvm/install)
2. [PacketFu, a mid-level packet manipulation library for Ruby](https://github.com/packetfu/packetfu)
3. [TCP/IP Ilustrated vol. I by Stevens, 1994](https://www.amazon.com/TCP-Illustrated-Vol-Addison-Wesley-Professional/dp/0201633469)
4. [Attacking network protocols by James Forshaw, 2018](https://www.amazon.com/Attacking-Network-Protocols-Analysis-Exploitation/dp/1593277504)
