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

Add conntrack to fix problem

```
# Add conntrack to iptables
user1@server:~$ sudo iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy DROP 15 packets, 1455 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   49  3156 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
   32  2288 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
 1689  118K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

user1@server:~$ ping -c 2 -W 2 10.0.2.20
PING 10.0.2.20 (10.0.2.20) 56(84) bytes of data.
64 bytes from 10.0.2.20: icmp_seq=1 ttl=64 time=1.67 ms
64 bytes from 10.0.2.20: icmp_seq=2 ttl=64 time=1.79 ms

--- 10.0.2.20 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 1.666/1.730/1.794/0.064 ms

user1@server:~$ curl -m 5 -I http://10.0.2.20:8000
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.14.4
Date: Tue, 17 Nov 2026 19:51:23 GMT
Content-type: text/html; charset=utf-8
Content-Length: 551


```

Server VM  PCAP

```
19:51:51.571801 IP 10.0.2.10.50586 > 10.0.2.20.8000: Flags [S], seq 2376772927, win 64240, options [mss 1460,sackOK,TS val 3042405113 ecr 0,nop,wscale 8], length 0
19:51:51.572520 IP 10.0.2.20.8000 > 10.0.2.10.50586: Flags [S.], seq 2167143019, ack 2376772928, win 65160, options [mss 1460,sackOK,TS val 2332689347 ecr 3042405113,nop,wscale 8], length 0
19:51:51.572617 IP 10.0.2.10.50586 > 10.0.2.20.8000: Flags [.], ack 1, win 251, options [nop,nop,TS val 3042405114 ecr 2332689347], length 0
19:51:51.573105 IP 10.0.2.10.50586 > 10.0.2.20.8000: Flags [P.], seq 1:80, ack 1, win 251, options [nop,nop,TS val 3042405115 ecr 2332689347], length 79
19:51:51.573645 IP 10.0.2.20.8000 > 10.0.2.10.50586: Flags [.], ack 80, win 255, options [nop,nop,TS val 2332689348 ecr 3042405115], length 0
19:51:51.576120 IP 10.0.2.20.8000 > 10.0.2.10.50586: Flags [P.], seq 1:156, ack 80, win 255, options [nop,nop,TS val 2332689350 ecr 3042405115], length 155
19:51:51.576418 IP 10.0.2.10.50586 > 10.0.2.20.8000: Flags [.], ack 156, win 251, options [nop,nop,TS val 3042405118 ecr 2332689350], length 0
19:51:51.576690 IP 10.0.2.10.50586 > 10.0.2.20.8000: Flags [F.], seq 80, ack 156, win 251, options [nop,nop,TS val 3042405118 ecr 2332689350], length 0
19:51:51.578661 IP 10.0.2.20.8000 > 10.0.2.10.50586: Flags [F.], seq 156, ack 81, win 255, options [nop,nop,TS val 2332689352 ecr 3042405118], length 0
19:51:51.578728 IP 10.0.2.10.50586 > 10.0.2.20.8000: Flags [.], ack 157, win 251, options [nop,nop,TS val 3042405120 ecr 2332689352], length 0
```

Client VM

```
# Confirm that the default DROP policy is still active
user1@client:~$ sudo nmap -sS -p 80,45678 10.0.2.10
[sudo: authenticate] Password:                                            
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-17 19:54 +0000
Nmap scan report for 10.0.2.10
Host is up (0.0014s latency).

PORT      STATE    SERVICE
80/tcp    filtered http
45678/tcp filtered eba
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 6.89 seconds
```

### Exercise 2: Inspect the connection table

Server VM

```
# Check conntrack table
user1@server:~$ sudo conntrack -L
tcp      6 431999 ESTABLISHED src=192.168.56.10 dst=192.168.56.1 sport=22 dport=54419 src=192.168.56.1 dst=192.168.56.10 sport=54419 dport=22 [ASSURED] mark=0 use=1
conntrack v1.4.9 (conntrack-tools): 1 flow entries have been shown.

# Live view of conntrack table during the curl request
user1@server:~$ sudo conntrack -E
[sudo: authenticate] Password:                                            
    [NEW] tcp      6 120 SYN_SENT src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 [UNREPLIED] src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216
 [UPDATE] tcp      6 60 SYN_RECV src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216
 [UPDATE] tcp      6 432000 ESTABLISHED src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216 [ASSURED]
 [UPDATE] tcp      6 120 FIN_WAIT src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216 [ASSURED]
 [UPDATE] tcp      6 60 CLOSE_WAIT src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216 [ASSURED]
 [UPDATE] tcp      6 30 LAST_ACK src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216 [ASSURED]
 [UPDATE] tcp      6 120 TIME_WAIT src=10.0.2.10 dst=10.0.2.20 sport=57216 dport=8000 src=10.0.2.20 dst=10.0.2.10 sport=8000 dport=57216 [ASSURED]

```
### Exercise 3: Why RELATED matters

Server VM

```
# list firewall rules with line numbers
user1@server:~$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy DROP 26 packets, 3234 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1     2569  188K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
2       48  3508 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
3     1690  118K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

# Delete the first firewall rule       
user1@server:~$ sudo iptables -D INPUT 1

# Show updated firewall rules
user1@server:~$ sudo iptables -n -L -v
Chain INPUT (policy DROP 26 packets, 3234 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   48  3508 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
 1765  123K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

# 
user1@server:~$ sudo iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED -j ACCEPT

user1@server:~$ sudo iptables -L -v -n
Chain INPUT (policy DROP 26 packets, 3234 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   51  3372 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
   48  3508 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
 1959  136K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

# test if not having RELATED matters
user1@server:~$ sudo nmap -sU -p 5555 10.0.2.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-18 16:04 +0000
Nmap scan report for server.local (10.0.2.10)
Host is up (0.000054s latency).

PORT     STATE  SERVICE
5555/udp closed rplay

Nmap done: 1 IP address (1 host up) scanned in 0.21 seconds

# packts were dropped but the state from nmap came back as "closed" instead of "open | filtered"

# to test that the packets are classified as RELATED, added new iptables rules
user1@server:~$ sudo iptables -I INPUT 1 -p icmp -m conntrack --ctstate RELATED

user1@server:~$ sudo iptables -I INPUT 1 -p icmp -m conntrack --ctstate ESTABLISHED

user1@server:~$ sudo iptables -Z

user1@server:~$ sudo nmap -sU -p 5555 10.0.2.20
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-18 16:13 +0000
Nmap scan report for 10.0.2.20
Host is up (0.0012s latency).

PORT     STATE  SERVICE
5555/udp closed rplay
MAC Address: 08:00:27:29:D6:FE (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 6.17 seconds
user1@server:~$ sudo iptables -L INPUT -n -v --line-numbers
Chain INPUT (policy DROP 1 packets, 56 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
2        1    56            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
3      376 24952 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
4        1    68 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
5        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
```

A more reliable test for RELATED

```
user1@server:~$ sudo iptables -I INPUT 1 -p icmp -m conntrack --ctstate RELATED

user1@server:~$ sudo iptables -Z

user1@server:~$ sudo iptables -L INPUT -n -v --line-numbers
Chain INPUT (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
2        8   476 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
3        0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
4        0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
5        0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
6        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
7        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

user1@server:~$ nc -uvz -w 3 10.0.2.20 5555

user1@server:~$ sudo iptables -L INPUT -n -v --line-numbers
Chain INPUT (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        2   114            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
2      142  9266 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
3        0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
4        0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
5        0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
6        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
7        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

# nc (netstat) packets were classified as RELATED (line 1)

# PCAPS also confirmed it
21:28:11.150935 IP 10.0.2.10.50682 > 10.0.2.20.5555: UDP, length 1
21:28:11.151055 IP 10.0.2.10.50682 > 10.0.2.20.5555: UDP, length 1
21:28:11.152073 IP 10.0.2.20 > 10.0.2.10: ICMP 10.0.2.20 udp port 5555 unreachable, length 37
21:28:11.152073 IP 10.0.2.20 > 10.0.2.10: ICMP 10.0.2.20 udp port 5555 unreachable, length 37

```

### Exercise 4: A complete stateful ruleset

#### Step 1: Clean up

Server VM

```
# Set default INPUT policy as ACCEPT
user1@server:~$ sudo iptables -P INPUT ACCEPT

# List iptables to show ACCEPT
user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    4   228            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
 1080 78234 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
    0     0            icmp --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate ESTABLISHED
   48  3576 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
    1    52 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

# Flush the iptable
user1@server:~$ sudo iptables -F

# Show empty iptable
user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 8 packets, 476 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination    
```
Client VM

```
# confirm http service is still running on port 8000
user1@client:~$ ss -tlnp
State      Recv-Q     Send-Q         Local Address:Port         Peer Address:Port    Process                                
LISTEN     0          4096           127.0.0.53%lo:53                0.0.0.0:*                                              
LISTEN     0          511                  0.0.0.0:80                0.0.0.0:*                                              
LISTEN     0          4096                 0.0.0.0:22                0.0.0.0:*                                              
LISTEN     0          4096              127.0.0.54:53                0.0.0.0:*                                              
LISTEN     0          5                    0.0.0.0:8000              0.0.0.0:*        users:(("python3",pid=1510,fd=3))     
LISTEN     0          511                     [::]:80                   [::]:*                                              
LISTEN     0          4096                    [::]:22                   [::]:*    
```

#### Step 2: Add rules first, then policy last

Server VM

```
user1@server:~$ sudo iptables -A INPUT -i lo -j ACCEPT
user1@server:~$ sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
user1@server:~$ sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
user1@server:~$ sudo iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
user1@server:~$ sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

user1@server:~$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 487 packets, 33196 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   80  5968 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
  767 51704 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate INVALID
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 ctstate NEW
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 ctstate NEW
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
        
user1@server:~$ sudo iptables -P INPUT DROP

user1@server:~$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1       80  5968 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2      893 59956 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
3        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate INVALID
4        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 ctstate NEW
5        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 ctstate NEW
6        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination      
```
#### Step 3: Test inbound traffic (from the client VM)

Client VM

```
user1@client:~$ sudo nmap -sS -p 22,80,45678 10.0.2.10
[sudo: authenticate] Password:                                            
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-18 22:59 +0000
Nmap scan report for 10.0.2.10
Host is up (0.010s latency).

PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
45678/tcp filtered eba
MAC Address: 08:00:27:3F:30:9E (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 7.10 seconds

# test ping
user1@client:~$ ping -c 3 10.0.2.10
PING 10.0.2.10 (10.0.2.10) 56(84) bytes of data.
64 bytes from 10.0.2.10: icmp_seq=1 ttl=64 time=4.45 ms
64 bytes from 10.0.2.10: icmp_seq=2 ttl=64 time=2.57 ms
64 bytes from 10.0.2.10: icmp_seq=3 ttl=64 time=2.53 ms

--- 10.0.2.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2045ms
rtt min/avg/max/mdev = 2.531/3.182/4.446/0.893 ms

# test port 80
user1@client:~$ curl -m 5 -I http://10.0.2.10
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 18 Nov 2026 23:00:08 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Mon, 09 Nov 2026 20:59:24 GMT
Connection: keep-alive
ETag: "6af2342c-267"
Accept-Ranges: bytes
```

#### Step 4: Test outbound traffic (leaving the server VM)

```
user1@server:~$ curl -m 5 -I http://10.0.2.20:8000
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.14.4
Date: Wed, 18 Nov 2026 23:03:45 GMT
Content-type: text/html; charset=utf-8
Content-Length: 551

user1@server:~$ ping -c 2 10.0.2.20
PING 10.0.2.20 (10.0.2.20) 56(84) bytes of data.
64 bytes from 10.0.2.20: icmp_seq=1 ttl=64 time=4.89 ms
64 bytes from 10.0.2.20: icmp_seq=2 ttl=64 time=9.75 ms

--- 10.0.2.20 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1087ms
rtt min/avg/max/mdev = 4.889/7.319/9.749/2.430 ms
```

#### Step 5: Read the counters

```
user1@server:~$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy DROP 2 packets, 88 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1       96  7088 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2     1280 87209 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
3        1    52 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate INVALID
4        1    44 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 ctstate NEW
5        2   104 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 ctstate NEW
6        1    84 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination   
```
