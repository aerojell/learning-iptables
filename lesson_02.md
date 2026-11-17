# Lesson 02: Basic allow and deny rules

## Concepts
rule order, default policy, DROP vs. REJECT.

## Prep

### Step 1: Check if the lab works

Check if client could reach server
```
user1@client:~$ ping -c 3 10.0.2.10
PING 10.0.2.10 (10.0.2.10) 56(84) bytes of data.
64 bytes from 10.0.2.10: icmp_seq=1 ttl=64 time=11.9 ms
64 bytes from 10.0.2.10: icmp_seq=2 ttl=64 time=6.05 ms
64 bytes from 10.0.2.10: icmp_seq=3 ttl=64 time=6.81 ms
```

Check which ports are open on the server

```
user1@server:~$ sudo ss -tlnp
[sudo: authenticate] Password:                                            
State     Recv-Q    Send-Q       Local Address:Port       Peer Address:Port   Process                                                                         
LISTEN    0         4096         127.0.0.53%lo:53              0.0.0.0:*       users:(("systemd-resolve",pid=591,fd=17))                                       
LISTEN    0         4096               0.0.0.0:22              0.0.0.0:*       users:(("sshd",pid=1051,fd=3),("systemd",pid=1,fd=108))                         
LISTEN    0         511                0.0.0.0:80              0.0.0.0:*       users:(("nginx",pid=1253,fd=5),("nginx",pid=1252,fd=5),("nginx",pid=1250,fd=5)) 
LISTEN    0         4096            127.0.0.54:53              0.0.0.0:*       users:(("systemd-resolve",pid=591,fd=19))                                       
LISTEN    0         4096                  [::]:22                 [::]:*       users:(("sshd",pid=1051,fd=4),("systemd",pid=1,fd=109))                         
LISTEN    0         511                   [::]:80                 [::]:*       users:(("nginx",pid=1253,fd=6),("nginx",pid=1252,fd=6),("nginx",pid=1250,fd=6)) 
```

Start listener

```
user1@server:~$ nc -l -k 45678 &
[2] 1440
```

Check firewall rules

```user1@server:~$ sudo iptables -L -v -n
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination       

```

## Exercise 1: Block Port 80

Server VM

```
# Set iptables to DROP port 80 traffic 
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 80 -j DROP

```

Client VM

```
user1@client:~$ curl -m 5 -I http://10.0.2.10
curl: (28) Connection timed out after 5006 milliseconds

user1@client:~$ sudo nmap -sS -p 22,80 10.0.2.10
[sudo: authenticate] Password:                                            
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-13 20:51 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0044s latency).

PORT   STATE    SERVICE
22/tcp open     ssh
80/tcp filtered http
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.73 seconds
```

Server VM
```
user1@server:~$ sudo iptables -L -v -n
[sudo: authenticate] Password:                                            
Chain INPUT (policy ACCEPT 1745 packets, 2131K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    7   388 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination



user1@server:~$ sudo nft list ruleset
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
                tcp dport 80 counter packets 7 bytes 388 drop
        }
}

```

Flush firewall rule after this excercise

```
user1@server:~$ sudo iptables -F

user1@server:~$ sudo iptables -L -v -n
Chain INPUT (policy ACCEPT 2325 packets, 2186K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

user1@server:~$ sudo nft list ruleset
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
        }
}
```

## Exercise 2: DROP vs REJECT

### Part A: REJECT with a TCP reset

Server VM


```
# Set iptables to REJECT port 80 traffic with TCP reset
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 80 -j REJECT --reject-with tcp-reset

user1@server:~$ sudo iptables -L -v -n
Chain INPUT (policy ACCEPT 2396 packets, 2192K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REJECT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 reject-with tcp-reset

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

user1@server
:~$ sudo nft list ruleset
# Warning: table ip filter is managed by iptables-nft, do not touch!
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
                tcp dport 80 counter packets 0 bytes 0 reject with tcp reset
        }
}
```

Client VM

```
user1@client:~$ time curl -m 10 -I http://10.0.2.10
curl: (7) Failed to connect to 10.0.2.10 port 80 after 4 ms: Could not connect to server

real    0m0.200s
user    0m0.023s
sys     0m0.096s


user1@client:~$ sudo nmap -sS -p 80 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-16 18:47 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0036s latency).

PORT   STATE  SERVICE
80/tcp closed http
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.36 seconds
```

### Part B: Watch the packets

Server VM

```
user1@server:~$ sudo tcpdump -ni enp0s3 port 80
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes

18:56:30.765066 IP 10.0.2.20.50892 > 10.0.2.10.80: Flags [S], seq 1816930442, win 64240, options [mss 1460,sackOK,TS val 1543390701 ecr 0,nop,wscale 8], length 0
18:56:30.765108 IP 10.0.2.10.80 > 10.0.2.20.50892: Flags [R.], seq 0, ack 1816930443, win 0, length 0

```

### Part C: Compare with DROP

Server VM

```
# Flush iptables
user1@server:~$ sudo iptables -F

# Confirm thee's nothing there using iptables and nft
user1@server:~$ sudo nft list ruleset
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
        }
}

user1@server:~$ sudo iptables -L -v -n
Chain INPUT (policy ACCEPT 2536 packets, 2197K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

# Set iptables to DROP port 80 traffic
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 80 -j DROP

```

Client VM

```
user1@client:~$ time curl -m 10 -I http://10.0.2.10
curl: (28) Connection timed out after 12066 milliseconds

real    0m12.135s
user    0m0.041s
sys     0m0.014s
```

Server VM

```
user1@server:~$ sudo tcpdump -ni enp0s3 port 80
[sudo: authenticate] Password:                                            
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:36:00.161620 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836444951 ecr 0,nop,wscale 8], length 0
19:36:01.279874 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836446069 ecr 0,nop,wscale 8], length 0
19:36:03.810679 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836448600 ecr 0,nop,wscale 8], length 0
19:36:08.600855 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836453390 ecr 0,nop,wscale 8], length 0
19:36:09.661759 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836454451 ecr 0,nop,wscale 8], length 0
19:36:12.260548 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836457050 ecr 0,nop,wscale 8], length 0
19:36:15.908263 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836460697 ecr 0,nop,wscale 8], length 0
19:36:22.155775 IP 10.0.2.20.60012 > 10.0.2.10.80: Flags [S], seq 2485536836, win 64240, options [mss 1460,sackOK,TS val 2836466945 ecr 0,nop,wscale 8], length 0
```

### Part D: Plain REJECT

```
# Flush existing iptables
user1@server:~$ sudo iptables -F

# Confirm there's no iptables 
user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 3702 packets, 2777K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

user1@server:~$ sudo nft list ruleset
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
        }
}

# Set iptables to "plain" REJECT port 80 traffic
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 80 -j REJECT

# Verify iptable rules
user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 4147 packets, 2814K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REJECT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

user1@server:~$ sudo nft list ruleset
# Warning: table ip filter is managed by iptables-nft, do not touch!
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
                tcp dport 80 counter packets 0 bytes 0 reject
        }
}
```

Client VM

```
user1@client:~$ sudo time curl -m 10 http://10.0.2.10
[sudo: authenticate] Password:                                            
curl: (7) Failed to connect to 10.0.2.10 port 80 after 4 ms: Could not connect to server
Command exited with non-zero status 7
0.01user 0.03system 0:00.09elapsed 48%CPU (0avgtext+0avgdata 13156maxresident)k
5480inputs+0outputs (11major+697minor)pagefaults 0swaps

user1@client:~$ sudo nmap -sS -p22,80 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-17 15:18 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0011s latency).

PORT   STATE    SERVICE
22/tcp open     ssh
80/tcp filtered http
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.02 seconds
```
Server VM
```
user1@server:~$ sudo tcpdump -ni enp0s3 port 80
[sudo: authenticate] Password:                                            
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:17:27.396899 IP 10.0.2.20.40688 > 10.0.2.10.80: Flags [S], seq 2770638832, win 64240, options [mss 1460,sackOK,TS val 4033555141 ecr 0,nop,wscale 8], length 0
```

## Exercise 3: Default DENY policy

### Prep: Clear existing firewall rules

Server VM

```
user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 410 packets, 45738 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    2   104 REJECT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

user1@server:~$ sudo iptables -F

user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 527 packets, 53330 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 
user1@server:~$ sudo nft list ruleset
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
        }
}

# start nc listener
user1@server:~$ nc -l -k 45678 &
[1] 1766
```

### Step 1: Add ALLOW rules first

Server VM

```
# Allow traffic from loopback interface (lo)
user1@server:~$ sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH traffic
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow ping request
user1@server:~$ sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

user1@server:~$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy ACCEPT 1032 packets, 94699 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2      490 32536 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
3        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination  

      
user1@server:~$ sudo nft list ruleset
# Warning: table ip filter is managed by iptables-nft, do not touch!
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
                iifname "lo" counter packets 0 bytes 0 accept
                tcp dport 22 counter packets 330 bytes 21708 accept
                ip protocol icmp icmp type echo-request counter packets 0 bytes 0 accept
        }
}
```

## Step 2: Flip policy to DROP

Server VM

```
user1@server:~$ sudo iptables -P INPUT DROP

user1@server:~$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2      779 52072 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
3        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

Client VM

```
# check ping
user1@client:~$ ping -c 3 10.0.2.10
PING 10.0.2.10 (10.0.2.10) 56(84) bytes of data.
64 bytes from 10.0.2.10: icmp_seq=1 ttl=64 time=3.70 ms
64 bytes from 10.0.2.10: icmp_seq=2 ttl=64 time=3.85 ms
64 bytes from 10.0.2.10: icmp_seq=3 ttl=64 time=5.59 ms

--- 10.0.2.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2442ms
rtt min/avg/max/mdev = 3.699/4.377/5.588/0.857 ms

# check ssh
user1@client:~$ ssh user1@10.0.2.10
The authenticity of host '10.0.2.10 (10.0.2.10)' can't be established.
ED25519 key fingerprint is: SHA256:FHIl/ZqoFQwtRaVdTpdCXGMlzwgCKRODHvYUlxEEI5w
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.10' (ED25519) to the list of known hosts.
user1@10.0.2.10's password: 
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-43-generic x86_64)

# check http
user1@client:~$ curl -m 5 -I http://10.0.2.10
curl: (28) Connection timed out after 5009 milliseconds

# run nmap
user1@client:~$ sudo nmap -sS -p- 10.0.2.10
[sudo: authenticate] Password:                                            
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-17 15:38 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0010s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 116.19 seconds

```

Server VM

```
# ping can't get make it out of the server
user1@server:~$ ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2152ms

# apt updates don't work either
user1@server:~$ sudo apt update
Ign:1 http://security.ubuntu.com/ubuntu resolute-security InRelease         
Ign:2 http://us.archive.ubuntu.com/ubuntu resolute InRelease                
Ign:1 http://security.ubuntu.com/ubuntu resolute-security InRelease         
Ign:3 http://us.archive.ubuntu.com/ubuntu resolute-updates InRelease        
Ign:4 http://us.archive.ubuntu.com/ubuntu resolute-backports InRelease      
Ign:1 http://security.ubuntu.com/ubuntu resolute-security InRelease         
Err:1 http://security.ubuntu.com/ubuntu resolute-security InRelease         
  Temporary failure resolving 'security.ubuntu.com'
Ign:2 http://us.archive.ubuntu.com/ubuntu resolute InRelease                
Ign:3 http://us.archive.ubuntu.com/ubuntu resolute-updates InRelease
Ign:4 http://us.archive.ubuntu.com/ubuntu resolute-backports InRelease
Ign:2 http://us.archive.ubuntu.com/ubuntu resolute InRelease
Ign:3 http://us.archive.ubuntu.com/ubuntu resolute-updates InRelease
Ign:4 http://us.archive.ubuntu.com/ubuntu resolute-backports InRelease
Err:2 http://us.archive.ubuntu.com/ubuntu resolute InRelease
  Temporary failure resolving 'us.archive.ubuntu.com'
Err:3 http://us.archive.ubuntu.com/ubuntu resolute-updates InRelease
  Temporary failure resolving 'us.archive.ubuntu.com'
Err:4 http://us.archive.ubuntu.com/ubuntu resolute-backports InRelease
  Temporary failure resolving 'us.archive.ubuntu.com'
All packages are up to date.            
Warning: Failed to fetch http://us.archive.ubuntu.com/ubuntu/dists/resolute/InRelease  Temporary failure resolving 'us.archive.ubuntu.com'
Warning: Failed to fetch http://us.archive.ubuntu.com/ubuntu/dists/resolute-updates/InRelease  Temporary failure resolving 'us.archive.ubuntu.com'
Warning: Failed to fetch http://us.archive.ubuntu.com/ubuntu/dists/resolute-backports/InRelease  Temporary failure resolving 'us.archive.ubuntu.com'
Warning: Failed to fetch http://security.ubuntu.com/ubuntu/dists/resolute-security/InRelease  Temporary failure resolving 'security.ubuntu.com'
Warning: Some index files failed to download. They have been ignored, or old ones used instead.
```
