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

```
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 80 -j REJECT --reject-with tcp-reset

user1@server:~$ sudo iptables -L -v -n
Chain INPUT (policy ACCEPT 2396 packets, 2192K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REJECT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 reject-with tcp-reset

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

user1@server:~$ sudo nft list ruleset
# Warning: table ip filter is managed by iptables-nft, do not touch!
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy accept;
                tcp dport 80 counter packets 0 bytes 0 reject with tcp reset
        }
}
```
