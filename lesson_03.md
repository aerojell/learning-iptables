# Stateful Filtering

## Concepts

Conntrack labels every packet with a state:

| State | Meaning |
|-------|---------|
| **NEW** | The first packet of a connection (a TCP SYN, for example) |
| **ESTABLISHED** | Part of a connection already seen in both directions, including replies |
| **RELATED** | A new connection tied to an existing one, such as an ICMP error about a connection that exists |
| **INVALID** | Doesn’t fit any known connection (stray or malformed packets) |

### Setup

Server VM

```
# Check for clean firewall
user1@server:~$ sudo iptables -L INPUT -v -n
[sudo: authenticate] Password:                                            
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

```

Client VM

```
# Make sure there's something on the client to connect to (in a separate 'client' terminal)
user1@client:~$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...

```

### Exercise 1: See the problem, then fix it

Server VM

```
# Allow loopback traffic
user1@server:~$ sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH connections
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Confirm via traffic stats
user1@server:~$ sudo iptables -n -L -v --line-numbers
Chain INPUT (policy ACCEPT 120 packets, 7868 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2      104  6932 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

# Set DROP as default policy
user1@server:~$ sudo iptables -P INPUT DROP

# Check iptables to confirm default DROP policy
user1@server:~$ sudo iptables -n -L -v --line-numbers
Chain INPUT (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2      204 13300 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

# Try to connect out using ping
user1@server:~$ ping -c 2 -W 2 10.0.2.20
PING 10.0.2.20 (10.0.2.20) 56(84) bytes of data.

--- 10.0.2.20 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1014ms

# Try using http to client
user1@server:~$ curl -m 5 -I http://10.0.2.20:8000
curl: (28) Connection timed out after 5002 milliseconds 
```

Server VM PCAP

```
user1@server:~$ sudo tcpdump -ni enp0s3 port 8000
[sudo: authenticate] Password:                                            
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:43:57.288390 IP 10.0.2.10.35364 > 10.0.2.20.8000: Flags [S], seq 1029596761, win 64240, options [mss 1460,sackOK,TS val 1128035291 ecr 0,nop,wscale 8], length 0
19:43:57.289233 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096749479 ecr 1128035291,nop,wscale 8], length 0
19:43:58.304415 IP 10.0.2.10.35364 > 10.0.2.20.8000: Flags [S], seq 1029596761, win 64240, options [mss 1460,sackOK,TS val 1128036307 ecr 0,nop,wscale 8], length 0
19:43:58.306273 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096750496 ecr 1128035291,nop,wscale 8], length 0
19:43:59.328270 IP 10.0.2.10.35364 > 10.0.2.20.8000: Flags [S], seq 1029596761, win 64240, options [mss 1460,sackOK,TS val 1128037331 ecr 0,nop,wscale 8], length 0
19:43:59.330782 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096751520 ecr 1128035291,nop,wscale 8], length 0
19:44:00.352456 IP 10.0.2.10.35364 > 10.0.2.20.8000: Flags [S], seq 1029596761, win 64240, options [mss 1460,sackOK,TS val 1128038355 ecr 0,nop,wscale 8], length 0
19:44:00.354202 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096752544 ecr 1128035291,nop,wscale 8], length 0
19:44:01.376345 IP 10.0.2.10.35364 > 10.0.2.20.8000: Flags [S], seq 1029596761, win 64240, options [mss 1460,sackOK,TS val 1128039379 ecr 0,nop,wscale 8], length 0
19:44:01.377233 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096753567 ecr 1128035291,nop,wscale 8], length 0
19:44:03.534013 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096755724 ecr 1128035291,nop,wscale 8], length 0
19:44:06.315557 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096758505 ecr 1128035291,nop,wscale 8], length 0
19:44:11.744947 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096763935 ecr 1128035291,nop,wscale 8], length 0
19:44:19.911882 IP 10.0.2.20.8000 > 10.0.2.10.35364: Flags [S.], seq 1851860441, ack 1029596762, win 65160, options [mss 1460,sackOK,TS val 3096772101 ecr 1128035291,nop,wscale 8], length 0
```
