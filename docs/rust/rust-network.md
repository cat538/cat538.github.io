---
title: rust-网络
cover: 'https://github.com/cat538/images-auto/raw/main/img/fire.png'
date: 2022-05-30 22:31:19
tags: ['rust']
categories:
mathjax:
---

标准库的`net` moudle提供了 TCP/UDP 连接的基本原语

- [`TcpListener`](https://doc.rust-lang.org/std/net/struct.TcpListener.html) 和 [`TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html) provide functionality for communication over TCP

- [`UdpSocket`](https://doc.rust-lang.org/std/net/struct.UdpSocket.html) provides functionality for communication over UDP

- [`IpAddr`](https://doc.rust-lang.org/std/net/enum.IpAddr.html) 表示IPV4或IPV6类型; [`Ipv4Addr`](https://doc.rust-lang.org/std/net/struct.Ipv4Addr.html) 和 [`Ipv6Addr`](https://doc.rust-lang.org/std/net/struct.Ipv6Addr.html) 是对应的 IPv4 和IPv6 地址

- [`SocketAddr`](https://doc.rust-lang.org/std/net/enum.SocketAddr.html) 表示 IPv4 或 IPv6 的socket地址; [`SocketAddrV4`](https://doc.rust-lang.org/std/net/struct.SocketAddrV4.html) and [`SocketAddrV6`](https://doc.rust-lang.org/std/net/struct.SocketAddrV6.html) 是对应的 IPv4 和IPv6 socket地址

- [`ToSocketAddrs`](https://doc.rust-lang.org/std/net/trait.ToSocketAddrs.html) 是一个用于把其它类型（比如`&str`，`SocketAddrV6`，`(IpAddr, u16)`，`(&str, u16)`等）转成 [`SocketAddr`](https://doc.rust-lang.org/std/net/enum.SocketAddr.html) 类型的trait，从而更好的与 [`TcpListener`](https://doc.rust-lang.org/std/net/struct.TcpListener.html), [`TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html) or [`UdpSocket`](https://doc.rust-lang.org/std/net/struct.UdpSocket.html) 等类型交互

  > [`TcpStream::connect`] is an example of an function that utilizes `ToSocketAddrs` as a trait bound on its parameter in order to accept different types:
  >
  > ```rust
  > use std::net::{TcpStream, Ipv4Addr};
  > // #[stable(feature = "rust1", since = "1.0.0")]
  > // pub fn bind<A: ToSocketAddrs>(addr: A) -> io::Result<TcpListener> {
  > //     super::each_addr(addr, net_imp::TcpListener::bind).map(TcpListener)
  > // }
  > let stream = TcpStream::connect(("127.0.0.1", 443)); // or
  > let stream = TcpStream::connect("127.0.0.1:443"); // or
  > let stream = TcpStream::connect((Ipv4Addr::new(127, 0, 0, 1), 443));
  > ```

- module中的其它类型是此module中的参数类型或返回值类型

**需要注意**：在可能的情况下，Rust默认禁用对子进程继承套接字对象。例如，通过在UNIX系统中使用`CLOEXEC`标志或在Windows中使用`HANDLE_FLAG_INHERIT`标志。

