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

### Scan 1: the default scan

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
### Scan 2: every port

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

### Scan 3: identify the services

```
user1@client:~$ sudo nmap -sV -p 22,80,45678 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 20:31 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0030s latency).

PORT      STATE  SERVICE VERSION
22/tcp    open   ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 (Ubuntu Linux; protocol 2.0)
80/tcp    open   http    nginx 1.28.3 (Ubuntu)
45678/tcp closed eba
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.43 seconds
```

### Scan 4: finds which hosts are up on the subnet without scanning ports (a ping sweep).

```
user1@client:~$ sudo nmap -sn 10.0.2.0/24
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 20:33 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0033s latency).
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)
Nmap scan report for client.local (10.0.2.20)
Host is up.
Nmap done: 256 IP addresses (2 hosts up) scanned in 2.67 seconds
```

### Scan 5: skips host discovery and scans anyway. You’ll need this later if a firewall blocks ping, because nmap otherwise concludes “host seems down.”

```
user1@client:~$ sudo nmap -Pn 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 20:35 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0018s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.13 seconds
```

### Scan 6: adds more detail while it runs

```
user1@client:~$ sudo nmap -v 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-12 20:36 +0000
Initiating ARP Ping Scan at 20:36
Scanning 10.0.2.10 [1 port]
Completed ARP Ping Scan at 20:36, 0.21s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 20:36
Completed Parallel DNS resolution of 1 host. at 20:36, 0.50s elapsed
Initiating SYN Stealth Scan at 20:36
Scanning 10.0.2.10 [1000 ports]
Discovered open port 22/tcp on 10.0.2.10
Discovered open port 80/tcp on 10.0.2.10
Completed SYN Stealth Scan at 20:36, 0.37s elapsed (1000 total ports)
Nmap scan report for 10.0.2.10
Host is up (0.0037s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.22 seconds
           Raw packets sent: 1001 (44.028KB) | Rcvd: 1001 (40.036KB)
```

## Exercise 2: tcpdump

Capture a full TCP handshake and a connection teardown with tcpdump.

### Step 1: Start the capture on server

```
user1@server:~$ sudo tcpdump -ni enp0s3 port 80
[sudo: authenticate] Password:         
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
```

### Step 2: Generate a connection from client

```
user1@client:~$ curl -I http://10.0.2.10
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Thu, 12 Nov 2026 16:29:38 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Mon, 09 Nov 2026 20:59:24 GMT
Connection: keep-alive
ETag: "6af2342c-267"
Accept-Ranges: bytes
```

### Step 3: Read the output

```
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:29:38.539471 IP 10.0.2.20.49226 > 10.0.2.10.80: Flags [S], seq 3534882120, win 64240, options [mss 1460,sackOK,TS val 3220893198 ecr 0,nop,wscale 8], length 0
16:29:38.539511 IP 10.0.2.10.80 > 10.0.2.20.49226: Flags [S.], seq 2181120779, ack 3534882121, win 65160, options [mss 1460,sackOK,TS val 3445215309 ecr 3220893198,nop,wscale 8], length 0
16:29:38.540851 IP 10.0.2.20.49226 > 10.0.2.10.80: Flags [.], ack 1, win 251, options [nop,nop,TS val 3220893200 ecr 3445215309], length 0
16:29:38.540851 IP 10.0.2.20.49226 > 10.0.2.10.80: Flags [P.], seq 1:75, ack 1, win 251, options [nop,nop,TS val 3220893200 ecr 3445215309], length 74: HTTP: HEAD / HTTP/1.1
16:29:38.540988 IP 10.0.2.10.80 > 10.0.2.20.49226: Flags [.], ack 75, win 255, options [nop,nop,TS val 3445215310 ecr 3220893200], length 0
16:29:38.544708 IP 10.0.2.10.80 > 10.0.2.20.49226: Flags [P.], seq 1:248, ack 75, win 255, options [nop,nop,TS val 3445215314 ecr 3220893200], length 247: HTTP: HTTP/1.1 200 OK
16:29:38.546497 IP 10.0.2.20.49226 > 10.0.2.10.80: Flags [.], ack 248, win 251, options [nop,nop,TS val 3220893205 ecr 3445215314], length 0
16:29:38.547519 IP 10.0.2.20.49226 > 10.0.2.10.80: Flags [F.], seq 75, ack 248, win 251, options [nop,nop,TS val 3220893206 ecr 3445215314], length 0
16:29:38.547945 IP 10.0.2.10.80 > 10.0.2.20.49226: Flags [F.], seq 248, ack 76, win 255, options [nop,nop,TS val 3445215317 ecr 3220893206], length 0
16:29:38.553577 IP 10.0.2.20.49226 > 10.0.2.10.80: Flags [.], ack 249, win 251, options [nop,nop,TS val 3220893212 ecr 3445215317], length 0

10 packets captured
10 packets received by filter
0 packets dropped by kernel
```

## Exercise 3:

List the current rules with iptables -L -v -n and nft list ruleset, and read what’s already there.

### Step 1: See which iptables you have

```
user1@server:~$ iptables --version
iptables v1.8.11 (nf_tables)
```

### Step 2: List the rules with iptables

```
user1@server:~$ sudo iptables -L -v -n
[sudo: authenticate] Password:         
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

