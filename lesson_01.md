# Lesson 01 - Mental Model and observation

## Goal
be able to say where in the packet’s path a firewall decision happens.

## Concepts
packets, ports, TCP handshake, stateless vs. stateful filtering, chains/hooks (input, output, forward).

## Exercise 1: nmap
Use nmap from the client to scan the server and list open ports.

### Setup server

> sudo systemctl enable --now nginx ssh
> nc -l -k -p 45678 &        # toy TCP service on port 9000
> sudo ss -tlnp             # lists what's listening, so you know what nmap *should* find

> user1@server:~$ sudo ss -tlnp
> [sudo: authenticate] Password:         
> State    Recv-Q    Send-Q       Local Address:Port        Peer Address:Port   Process                                                                            
> LISTEN   0         511                0.0.0.0:80               0.0.0.0:*       users:(("nginx",pid=1258,fd=5),("nginx",pid=1257,fd=5),("nginx",pid=1256,fd=5))   
> LISTEN   0         4096               0.0.0.0:22               0.0.0.0:*       users:(("sshd",pid=1053,fd=3),("systemd",pid=1,fd=108))                           
> LISTEN   0         4096            127.0.0.54:53               0.0.0.0:*       users:(("systemd-resolve",pid=592,fd=19))                                         
> LISTEN   0         4096         127.0.0.53%lo:53               0.0.0.0:*       users:(("systemd-resolve",pid=592,fd=17))                                         
> LISTEN   0         1                  0.0.0.0:45678            0.0.0.0:*       users:(("nc",pid=1431,fd=3))                                                      
> LISTEN   0         511                   [::]:80                  [::]:*       users:(("nginx",pid=1258,fd=6),("nginx",pid=1257,fd=6),("nginx",pid=1256,fd=6))   
> LISTEN   0         4096                  [::]:22                  [::]:*       users:(("sshd",pid=1053,fd=4),("systemd",pid=1,fd=109)) 

