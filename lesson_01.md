# Lesson 01 - Mental Model and observation

## Goal
be able to say where in the packet’s path a firewall decision happens.

## Concepts
packets, ports, TCP handshake, stateless vs. stateful filtering, chains/hooks (input, output, forward).

## Exercise 1: nmap
Use nmap from the client to scan the server and list open ports.

### Setup server

```
  sudo systemctl enable --now nginx ssh
  nc -l -k -p 45678 &        # toy TCP service on port 9000
  sudo ss -tlnp              # lists what's listening, so you know what nmap *should* find
```

*Initially, Claude recommend using port 9000 for nc but realized that it's one of the known ports so it didn't show up in the initial nmap scan*

```
user1@server:~$ sudo ss -tlnp
[sudo: authenticate] Password:         
State    Recv-Q    Send-Q       Local Address:Port        Peer Address:Port   Process                                                                            
LISTEN   0         511                0.0.0.0:80               0.0.0.0:*       users:(("nginx",pid=1258,fd=5),("nginx",pid=1257,fd=5),("nginx",pid=1256,fd=5))   
LISTEN   0         4096               0.0.0.0:22               0.0.0.0:*       users:(("sshd",pid=1053,fd=3),("systemd",pid=1,fd=108))                           
LISTEN   0         4096            127.0.0.54:53               0.0.0.0:*       users:(("systemd-resolve",pid=592,fd=19))                                         
LISTEN   0         4096         127.0.0.53%lo:53               0.0.0.0:*       users:(("systemd-resolve",pid=592,fd=17))                                         
LISTEN   0         1                  0.0.0.0:45678            0.0.0.0:*       users:(("nc",pid=1431,fd=3))                                                      
LISTEN   0         511                   [::]:80                  [::]:*       users:(("nginx",pid=1258,fd=6),("nginx",pid=1257,fd=6),("nginx",pid=1256,fd=6))   
LISTEN   0         4096                  [::]:22                  [::]:*       users:(("sshd",pid=1053,fd=4),("systemd",pid=1,fd=109))
```

### Client Output

```
user1@client:~$ nmap 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 15:49 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0016s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

```
user1@client:~$ sudo nmap -sS -p- 10.0.2.10
[sudo: authenticate] Password:         
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 15:49 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0012s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
45678/tcp open  eba
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 12.53 seconds
```

*After this scan, I was curious why port 53 did not show up in the scans above:*

*- Claude told me that 53 is bound to 127.0.0.x so this is only accessed through loopback.*

*- The scan only looks for TCP ports and not UDP.  I re-run the scan below looking for UDP (see below)*

```
user1@client:~$ sudo nmap -sU -p 53 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 15:56 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0019s latency).

PORT   STATE  SERVICE
53/udp closed domain
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.87 seconds
```

## Exercise 2: tcpdump

