---
layout: post
title:  "Writing a Linux security sensor with eBPF and Ruby"
date:   2022-08-20 00:30:00
tags: c ruby ebpf security linux
---

eBPF is a technology that can run sandboxed programs in the operating system 
kernel and that has been incorporated into some security Linux sensors in
products for different vendors.

# What is eBPF?

The Berkeley Packet Filter (BPF) is a Linux kernel subsystem that allows a user to 
run a limited set of instructions on a virtual machine running in the kernel. 

It is divided between classic BPF (cBPF) and extended BPF (eBPF, or simply BPF). 

* cBPF is limited for the packet observing.
* eBPF is much more flexible and powerful than cBPF and allows the following actions: 
  * Modify packets 
  * Change syscall arguments
  * Modify userspace applications
  * Other operations

# How is eBPF used?

eBPF allows safely and efficiently extend some kernel capabilities 
without changes for the kernel source code or loading the kernel modules. 

Application developers can execute eBPF programs to add the operating system extra 
capabilities at runtime because eBPF permits running sandboxed applications inside 
the kernel without changes for the kernel source code or loading the kernel modules.
The operating system ensures security and performance as if natively developed using 
the Just-In-Time (JIT) compiler and verification engine for the eBPF subsystem.

![eBPF](/img/linux_security_sensor_with_ebpf/eBPF-diagram.png)

eBPF is widely used for the following use cases:

* Networking: Providing high-performance networking and load-balancing in data centers
and cloud environments.
* Security: Watching and understanding all system calls at the packet and socket levels 
of all network traffic operations to secure systems.
* Observability & Monitoring: Extracting fine-grained security observability data at
low overhead to enables the collection & in-kernel aggregation of custom metrics and
generation of visibility events based on a wide range of possible sources.
* Tracing & Profiling: Attaching eBPF programs to trace kernel and user applications 
allowing unprecedented visibility into the runtime behavior of applications and the 
* operating system.

The eBPF infrastructure includes by the following elements:

* eBPF Runtime: The Linux Kernel contains the eBPF runtime required to run eBPF instructions.
* LLVM/GCC Compiler: The compiler infrastructure contains the eBPF backend to translate
programs written in a C-like syntax to eBPF instructions.
* eBPF toolkits and libraries in different programming languages:
  * Go
  * C/C++
  * Rust
  * Ruby
  * Python

I will implement a proof of concept of a Linux security sensor to monitor the DNS traffic 
and identify malicious domains in a simulated compromised host. This sensor will use the 
following components:

* The C-like eBPF runtime instructions to filter the DNS packets from all the network traffic.
* The Ruby code to instrument the eBPF instructions by the BCC toolkit and parse the DNS packets 
to JSON and send them to a async-queue.
* An async listener that consumes the queue incoming messages and search for the domain names at 
an indicator of compromises database.
* if the host is malicious, the kernel blocks the traffic from and to that host.

# DNS packet matcher for eBPF
The following C code allows matching the DNS packets from all the network traffic using
the BCC kernel instrumentation toolkit for eBPF programs and bindings for 
multiple programming languages.

The dns_packet_matcher function uses the [__sk_buff](https://github.com/iovisor/bcc/blob/5bf9b4d145eeda9dc304c71212859ca9f3ebb3b9/src/cc/compat/linux/virtual_bpf.h#L5746) structure
provided by BCC, which is user accessible mirror of the in-kernel [sk_buff](https://www.kernel.org/doc/htmldocs/networking/API-struct-sk-buff.html) structure
and is used get access to the socket buffer members of each network packet.

This function also uses the [cursor_advance](https://github.com/iovisor/bcc/blob/master/src/cc/export/helpers.h#L524) macro 
to successively parse the different headers in each packet and then only match the UDP packets
with the 53 destination port.

This function will be instrumented in the kernel via the BCC toolkit and will be called by the frontend 
program that attaches this to a specific network interface.

{% highlight c linenos %}
#include <bcc/proto.h>

#define IP_UDP  0x11 // UDP Protocol: 17
#define DPORT   0x35 // Default DNS Destination Port: 53

struct dns_header_t
{
    uint16_t id;
    uint16_t flags;
    uint16_t nqueries;
    uint16_t nanswers;
    uint16_t nauth;
    uint16_t nother;
    unsigned char	data[1];
} BPF_PACKET_HEADER;

int dns_packet_matcher(struct __sk_buff *skb)
{
  u8 *cursor = 0;

  struct ethernet_t *ethernet = cursor_advance(cursor, sizeof(*ethernet));
  if(ethernet->type != ETH_P_IP) {
    return 0;
  }

  struct ip_t *ip = cursor_advance(cursor, sizeof(*ip));
  u16 hlen_bytes = ip->hlen << 2;
  if(ip->nextp != IP_UDP) {
    return 0;
  }

  struct udp_t *udp = cursor_advance(cursor, sizeof(*udp));
  if (udp->dport != DPORT){
    return 0;
  }

  struct dns_header_t *dns_header = cursor_advance(cursor, sizeof(*dns_header));
  if((dns_header->flags >>15) != 0) {
    return 0;
  }

  return -1;
} 
{% endhighlight %}
