---
title: "Which WireGuard on FreeBSD 13.1?"
date: 2022-06-04T00:00:00-06:00
---

I'm enjoying using FreeBSD 13.1 on my old ThinkPad X230.
One thing I haven't tried yet is using WireGuard for VPN connections on FreeBSD.
There are two WireGuard ports: `wireguard-kmod`, the experimental in-kernel version, and `wireguard-go`, the stable, cross-platform implementation.
I tried both to see which I should use.

<!--more-->

### Setting Up Experimental In-Kernel Wireguard

Let's start with the fun version.
This port is the in-kernel version from zx2c4, written by Jason Donenfeld and Kyle Evans--not to be confused with the infamous Netgate-sponsored port that was yanked from FreeBSD 13.
It's still under development and comes with disclaimers about using it in production.

To use it, install the package and the userland tools to control it.

```console
# pkg install wireguard-kmod wireguard-tools
```

The package notes (use `pkg info -D wireguard-kmod` to see them again) show a warning that the code is unvetted and might contain security issues.
They don't include information about how to use the kernel module.

I assumed I would have to load it by name with `kldload`.
To find the name, I listed all the files installed by the package.

```console
$ pkg info -l wireguard-kmod

wireguard-kmod-0.0.20211105_1:
	/boot/modules/if_wg.ko
	/usr/local/share/licenses/wireguard-kmod-0.0.20211105_1/LICENSE
	/usr/local/share/licenses/wireguard-kmod-0.0.20211105_1/MIT
	/usr/local/share/licenses/wireguard-kmod-0.0.20211105_1/catalog.mk
```

However, I got an error when I loaded it by name.

```console
# kldload if_wg
kldload: can't load if_wg: module already loaded or in kernel
```

Checking whether the module is loaded gives us exit code 0 (success).
I guess it's loaded by default!

```console
# kldstat -q -n if_wg; echo $?
0
```

Configuring, starting, and stopping tunnels works the same as on other platforms with `wireguard-tools`.
I defined the `wg0` tunnel by creating `/usr/local/etc/wireguard/wg0.conf`.

```toml
#
# wg0.conf - wireguard configuration for a client machine (e.g. a laptop)
#

[Interface]
Address = # 10.2.0.<Your Address Here>/24
PrivateKey = # <Your Private Key Here>
DNS = 1.1.1.1

# Gateway server
[Peer]
Endpoint = url.of.gateway:51820
PublicKey = # <Gateway Server's Public Key Here>
AllowedIPs = 10.2.0.0/24,192.168.1.0/24
PersistentKeepAlive = 25
```

Starting the tunnel revealed another difference between FreeBSD and Linux.

```console
# wg-quick up wg0           
[#] ifconfig wg create name wg0
[#] wg setconf wg0 /dev/stdin
[#] ifconfig wg0 inet 10.2.0.10/24 alias
[#] ifconfig wg0 mtu 1420
[#] ifconfig wg0 up
[#] resolvconf -a wg0 -x
[#] route -q -n add -inet 192.168.1.0/24 -interface wg0
[#] resolvconf -d wg0
[#] ifconfig wg0 destroy

```

Uh oh, `ifconfig` destroyed my wg0 interface immediately after creating it!
Apparently, FreeBSD won't let me add a route that conflicts with an already active interface.
Linux lets you create the link.

To set up a route for 192.168.1.0/24 (my home network subnet), I have to make sure I don't already have one.
For now, I only need to get to the 10.2.0.10/24 subnet, so I removed the other one from the AllowedIPs parameter.

#### `/usr/local/etc/wireguard/wg0.conf`

```toml
AllowedIPs = 10.2.0.0/24
```

That changed fixed the issue.

```console
# wg-quick up wg0
[#] ifconfig wg create name wg0
[#] wg setconf wg0 /dev/stdin
[#] ifconfig wg0 inet 10.2.0.10/24 alias
[#] ifconfig wg0 mtu 1420
[#] ifconfig wg0 up
[#] resolvconf -a wg0 -x
[+] Backgrounding route monitor

# wg show
interface: wg0
  public key: <My client's public key>
  private key: (hidden)
  listening port: 40839

peer: <My gateway server's public key>
  endpoint: <IP Address>:51820
  allowed ips: 10.2.0.0/24
  latest handshake: 30 seconds ago
  transfer: 92 B received, 212 B sent
  persistent keepalive: every 25 seconds
```

The last thing to do is to enable the tunnel on startup.
I suspected that wireguard-tools came with an `rc` script to do so.

```console
$ pkg info -l wireguard-tools
wireguard-tools-1.0.20210914_1:
	/usr/local/bin/wg
	/usr/local/bin/wg-quick
	/usr/local/etc/rc.d/wireguard
[...]
```

Yep, there it is. The comment at the top of `/usr/local/etc/rc.c/wireguard` explains how to configure it in `/etc/rc.conf`.

```sh
wireguard_enable="YES"
wireguard_interfaces="wg0"
```

With those lines, the tunnel can be started using `service wireguard start`.

### Setting Up Stable Userland WireGuard

All we have to do to try out `wireguard-go` is to swap out the package that's installed.
Configuring, starting, and stopping tunnels is handled by `wireguard-tools` and works exactly the same way with both versions.

```console
# pkg remove wireguard-kmod
[...]
# pkg install wireguard-go
```

Now, running `service wireguard start` shows this:

```console
# service wireguard start
[#] ifconfig wg create name wg0
[!] Missing WireGuard kernel support (ifconfig: SIOCIFCREATE2:
Invalid argument). Falling back to slow userspace implementation.
[#] wireguard-go wg0
┌──────────────────────────────────────────────────────┐
│                                                      │
│   Running wireguard-go is not required because this  │
│   kernel has first class support for WireGuard. For  │
│   information on installing the kernel module,       │
│   please visit:                                      │
│         https://www.wireguard.com/install/           │
│                                                      │
└──────────────────────────────────────────────────────┘
[...]
```

This is a bit of a mixed message.
With `wireguard-kmod`, I got a scary warning about it being experimental, but `wireguard-go` says it's OBE.
I want to use the in-kernel version for better performance, but I also want security and stability.
Let's see if the slow userspace implementation is really all that slow.

### FreeBSD Wireguard Shootout

As an unscientific test, I set up a tunnel between my laptop with FreeBSD 13.1 and my Raspberry Pi both connected to the same switch over gigabit ethernet.
I ran `iperf3` to measure the throughput and `vmstat` to see CPU and memory usage.
Here are the network throughput results.

#### Baseline (No Tunnel)

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.09 GBytes   935 Mbits/sec    0   sender
[  5]   0.00-10.04  sec  1.09 GBytes   932 Mbits/sec        receiver
```

#### `wireguard-go`

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   661 MBytes   555 Mbits/sec  423   sender
[  5]   0.00-10.02  sec   661 MBytes   553 Mbits/sec        receiver
```

#### `wireguard-kmod`

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   787 MBytes   660 Mbits/sec  138   sender
[  5]   0.00-10.03  sec   786 MBytes   658 Mbits/sec        receiver
```

Compare the CPU usage stats from `vmstat`.
Each line is a half-second interval from the middle of the iperf test.

#### Baseline (No tunnel)

```
cpu
sy     cs    us sy id
97596  40992  1 12 87
96434  40968  3 11 87
98020  40760  4 12 84
```

#### `wireguard-go`

```
cpu
sy     cs    us sy id
210358 85010 17 16 67
221720 89424 21 15 63
215976 87518 12 20 68
```

#### `wireguard-kmod`

```
cpu
sy    cs    us sy id
14908 41894  1 27 72
13274 35634  2 27 71
14110 37270  1 29 70
````

The number of system calls (`sy`) is the reason to have wireguard in the kernel instead of userland.
The userland port made the most system calls to transfer the least data.
The idle number (`id`) also shows a difference big enough for a user to notice; the in-kernel version uses a bit less CPU time overall.

### On Linux?

Just for fun, I booted a live USB stick of Fedora Linux, which has WireGuard in the kernel, and tried the same test.

#### Baseline (No Tunnel) on Linux

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.09 GBytes   936 Mbits/sec    0   sender
[  5]   0.00-10.00  sec  1.09 GBytes   934 Mbits/sec        receiver
```

#### `wireguard` on Linux

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   717 MBytes   601 Mbits/sec   78   sender
[  5]   0.00-10.00  sec   715 MBytes   600 Mbits/sec        receiver

```

### Conclusions

The goal of using WireGuard is to have a secure VPN connection with very little overhead.
In this test, `wireguard-go` on FreeBSD doesn't really deliver that.
If I were happy cutting network performance almost in half, I might as well use a different protocol.

On the other hand, `wireguard-kmod` clearly can perform better, even in casual use.
It performs about as well as the real competition, WireGuard on Linux.
Given that it's been over a year since the in-kernel WireGuard kerfuffle, I'll wager it's secure enough to use.

