% Hands-On Firewall Lab: iptables Reference

**Lessons 1 to 3, with corrected exercises, answer key, and command reference**

Revision 1, 2026-11-18. Lab: Ubuntu Server VMs in VirtualBox on a Windows 11 host.

This is a living reference. Exercises are marked **Verified** when you have run them in your lab and the results matched, **Corrected** when a lab result showed my first version was wrong, and **Not yet run** when they are written but untested here. The revision log at the end lists every change.

[TOC]

# 1. Lab setup

## 1.1 Layout

Three Ubuntu Server VMs (22.04 or 24.04 LTS), about 1 vCPU, 1 to 2 GB RAM and 10 GB disk each.

| VM | Role | Network adapters |
|---|---|---|
| `client` | Traffic source, scanner | Internal Network `labnet` (Lessons 1 to 4) or `netA` (Lessons 5, 7) |
| `server` | Web, SSH, toy services, the machine you firewall | Internal Network `labnet` or `netB` |
| `gateway` | Router/firewall (Lessons 5 and 7 only) | `netA`, `netB`, plus NAT for internet |

Addresses used in this guide: `server` = `10.0.2.10`, `client` = `10.0.2.20`, both `/24`.

## 1.2 Build steps (VirtualBox)

1. Check virtualization is enabled (Task Manager, Performance, CPU: "Virtualization: Enabled"). Hyper-V or WSL2 running alongside VirtualBox can make VMs slow.
2. Install VirtualBox from virtualbox.org and Ubuntu Server from ubuntu.com.
3. Create `ubuntu-base`: 1 to 2 GB RAM, 10 GB disk, tick **Skip Unattended Installation**. Before first start, open **Settings, Network, Adapter 1** and confirm **Attached to: NAT** (this is a per-VM setting inside VirtualBox, not a Windows setting).
4. In the installer, tick **Install OpenSSH server**.
5. After install, run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nftables iptables ufw tcpdump nmap netcat-openbsd curl conntrack nginx
sudo poweroff
```

6. Take a snapshot named `base-tools` (select the VM, open the **Snapshots** tool at the top right, click **Take**).
7. Clone twice (Full clone, **generate new MAC addresses**) to make `client` and `server`. If the clone fails with "could not create directory", set the clone **Path** explicitly (for example to a folder on your external drive), check the drive is writable and NTFS or exFAT, and see section 6.
8. For each clone, set **Adapter 1 to Internal Network** with the same name (for example `labnet`).

## 1.3 VirtualBox network modes

| Mode | VM to internet | Host and VM | VM and VM |
|---|---|---|---|
| NAT | Yes | Only via port forwarding | No |
| NAT Network | Yes | Only via port forwarding | Yes (same named network) |
| Bridged | Yes | Yes | Yes, and your whole LAN can reach the VM |
| Internal Network | No | No | Yes (same name only) |
| Host-only | No | Yes | Yes |
| Not attached | No | No | No |

Use **NAT** while building and updating, **Internal Network** for the lab, and optionally **Host-only** as a second adapter if you want to SSH in from Windows. Avoid Bridged for this lab.

If a clone still has a working NAT Adapter 2, it can reach the internet and gets a second interface and possibly a default route. During the exercises, untick **Cable Connected** under **Adapter 2, Advanced** (you can change it while the VM runs) and reconnect it only when you need updates.

## 1.4 Static IPs with netplan

On each clone, from the VirtualBox console (paste does not work there):

```bash
ip -br a                 # find the interface name, often enp0s3
sudo hostnamectl set-hostname server        # or client
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                 # use YOUR interface name
      addresses: [10.0.2.10/24]     # client: 10.0.2.20/24
```

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
ping -c 3 10.0.2.10      # from client
```

Common netplan problems:

- **"Cannot find unique matching interface for enp0s3":** the interface has a different name on this VM, or a leftover file (such as `00-installer-config.yaml`) has a `match: macaddress:` block that no longer matches the clone's new MAC. Check `ip -br a` and `sudo cat /etc/netplan/*.yaml`. Keep one file with only your settings and move the others to `/root/netplan-backup/`.
- **"Permissions ... are too open":** run `sudo chmod 600 /etc/netplan/*.yaml`.
- **DHCP plus static address:** netplan merges all `.yaml` files alphabetically, so a leftover `dhcp4: true` can stay active alongside your static address, and boot may pause waiting for DHCP on an internal network.

## 1.5 Working habits

- **Do firewall work from the VirtualBox console, not SSH.** A bad rule can lock out SSH, but the console does not depend on the network.
- **iptables rules live in kernel memory.** They vanish on reboot, which is a safety net while learning.
- **Snapshot before each lesson.** Power off first for a baseline snapshot (taken with the firewall clean). Live snapshots are fine for quick checkpoints, but restoring one brings back the loaded rules and running processes.
- **Check the baseline at the start of every session:**

```bash
sudo iptables -L -v -n
```

You want all three chains `policy ACCEPT` with no rules. If not:

```bash
sudo iptables -P INPUT ACCEPT      # policy first, then flush
sudo iptables -F
```

- **Optional SSH from Windows:** while an adapter is NAT, add a port-forward rule (Adapter 1, Advanced, Port Forwarding: TCP, Host IP `127.0.0.1`, Host Port `2222`, Guest Port `22`, Guest IP blank) and run `ssh -p 2222 user@127.0.0.1`. Give each VM a different host port. Once on Internal Network, add a Host-only adapter instead. Shared clipboard needs a desktop session, so it does not work on Ubuntu Server.

# 2. Lesson 1: Mental model and observation

**Concepts:** packets and ports, the TCP handshake, stateless versus stateful filtering, chains and hooks (input, output, forward).

**Before you start:** on `server`, start some services.

```bash
sudo systemctl enable --now nginx ssh
nc -l -k 45678 &          # toy TCP listener on an unusual port
sudo ss -tlnp             # confirm what is listening
```

## Exercise 1: Scan the server with nmap (Corrected)

Run from `client` against `server`.

```bash
nmap 10.0.2.10                          # default: top 1000 TCP ports
sudo nmap -sS -p- 10.0.2.10             # all 65,535 ports, SYN scan
sudo nmap -sV -p 22,80,45678 10.0.2.10  # identify the software
```

Expected: the default scan shows 22 and 80 only; the `-p-` scan adds 45678.

**Correction:** my first version used port 9000 for the toy listener and said a default scan would miss it. Port 9000 is in nmap's top-1000 list, so the default scan found it. Use a port well outside the common list, such as 45678, to show that a default scan can miss services.

**Reading nmap port states:**

| State | Meaning |
|---|---|
| open | Something is listening and replied |
| closed | Reachable, but nothing listening (host sent a reset) |
| filtered | No answer, usually a firewall silently dropping packets |
| open or filtered | nmap cannot tell (common with UDP) |

When many ports share a state, nmap prints one summary line (for example `Not shown: 65534 filtered tcp ports (no-response)`) and lists only the different ports. A port can therefore be counted as filtered without being named. Scan it alone (`-p 45678`) to see it listed.

**Why port 53 does not show up:** `systemd-resolved` listens on `127.0.0.53:53` and `127.0.0.54:53`. Those are loopback addresses, reachable only from the server itself, so a scan from `client` cannot see them. Where a service is bound decides who can reach it, before any firewall rule applies:

| Bound to | Reachable from |
|---|---|
| `127.0.0.1`, `127.0.0.53` | Only the machine itself |
| `10.0.2.10` | Anything that can reach that address |
| `0.0.0.0` or `[::]` | Every interface |

**Watch the scan from the other side** (on `server`):

```bash
sudo tcpdump -ni enp0s3 'tcp[tcpflags] & tcp-syn != 0'
```

You see SYNs from the client, answered with SYN-ACK (open) or RST (closed).

## Exercise 2: Capture a TCP handshake and teardown (Verified)

Two windows. On `server`:

```bash
sudo tcpdump -ni enp0s3 port 80
```

On `client`:

```bash
curl -I http://10.0.2.10
```

Stop the capture with Ctrl+C. Typical output (numbers will differ):

```
IP 10.0.2.20.51234 > 10.0.2.10.80: Flags [S],  seq 1000, length 0
IP 10.0.2.10.80 > 10.0.2.20.51234: Flags [S.], seq 5000, ack 1001, length 0
IP 10.0.2.20.51234 > 10.0.2.10.80: Flags [.],  ack 5001, length 0
IP 10.0.2.20.51234 > 10.0.2.10.80: Flags [P.], seq 1:79, ack 1, length 78: HTTP: HEAD / HTTP/1.1
IP 10.0.2.10.80 > 10.0.2.20.51234: Flags [P.], seq 1:250, ack 79, length 249: HTTP/1.1 200 OK
IP 10.0.2.20.51234 > 10.0.2.10.80: Flags [F.], seq 79, ack 250, length 0
IP 10.0.2.10.80 > 10.0.2.20.51234: Flags [F.], seq 250, ack 80, length 0
IP 10.0.2.20.51234 > 10.0.2.10.80: Flags [.],  ack 251, length 0
```

| Flag shown | Meaning |
|---|---|
| `[S]` | SYN |
| `[S.]` | SYN-ACK (the dot means ACK is set) |
| `[.]` | ACK |
| `[P.]` | PSH+ACK (data) |
| `[F.]` | FIN+ACK (closing) |
| `[R]` or `[R.]` | RST (reset or refusal) |

Packets 1 to 3 are the handshake, 4 and 5 are the HTTP request and response, and 6 to 8 are the teardown. Teardown may show three or four packets, which is normal.

Tips: `-S` shows absolute sequence numbers (relative is the default); `-w file.pcap` saves a capture (the file always stores absolute values); `-c 20` stops after 20 packets; `-X` shows hex and ASCII.

Optional: run `curl http://10.0.2.10:8080` with `port 8080` in the capture filter to see what a closed port looks like.

## Exercise 3: Read the current rules (Verified)

Read-only, on `server`.

```bash
iptables --version              # expect (nf_tables) on current Ubuntu
sudo iptables -L -v -n
sudo nft list ruleset           # may print nothing on a fresh VM
sudo ufw status                 # expect: inactive
```

Fresh output:

```
Chain INPUT (policy ACCEPT 1520 packets, 98K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
Chain OUTPUT (policy ACCEPT 1210 packets, 120K bytes)
```

| Chain | Traffic |
|---|---|
| INPUT | Destined for this machine |
| OUTPUT | Created by this machine |
| FORWARD | Passing through to somewhere else (only matters on a router) |

On current Ubuntu, `iptables` is the `iptables-nft` variant: it translates your commands into nftables rules in the same kernel nf_tables ruleset, so `iptables -L` and `nft list ruleset` are two views of one set of rules. (Older "legacy" iptables used separate kernel tables whose rules would not appear in `nft`.) Try it: add `sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT`, then run `sudo nft list ruleset`, then clean up with `sudo iptables -F`.

## Lesson 1 checkpoint questions and answers

**Exercise 1**

1. *What is the difference between the default scan and `-p-` for a port like 45678?* The default scan checks only the 1000 most common ports, so it misses a listener on an unusual port; `-p-` checks all 65,535.
2. *What does the server send back for an open port compared with a closed one?* SYN-ACK for open, RST for closed.
3. *What would nmap show if the server silently dropped packets to port 80?* `filtered`, because no reply comes back.

**Exercise 2**

1. *Which side sends the first FIN?* Whichever side closes first. With `curl -I`, curl exits after the response, so the client usually sends it. Check the source address of the first `[F.]` in your own capture.
2. *What do seq and ack do in the handshake?* Each side's ack is the other side's seq plus 1, because a SYN consumes one sequence number. Each side starts from its own random initial sequence number.
3. *What happens against a closed port?* The server answers the SYN immediately with a reset, and curl reports "Connection refused".

**Exercise 3**

1. *Which chain does a packet from `client` to `server` traverse on the server?* INPUT. The server's replies go through its OUTPUT chain.
2. *What is the default policy on each chain?* ACCEPT on all three, so with no rules nothing is filtered.
3. *Why do `iptables -L` and `nft list ruleset` show the same rule?* `iptables` here is a front end that translates into the kernel's nf_tables ruleset, so both commands read the same rules.

# 3. Lesson 2: Basic allow and deny rules

**Concepts:** rule order, first match wins, default policy, DROP versus REJECT.

**Before you start:** snapshot both VMs; work on `server` from the console; confirm a clean baseline (section 1.5). Useful commands:

```bash
sudo iptables -L INPUT -v -n --line-numbers   # rules, counters, rule numbers
sudo iptables -F                              # flush rules (does NOT reset the policy)
sudo iptables -P INPUT ACCEPT                 # reset the default policy
```

## Exercise 1: Block port 80 (Verified)

On `server`:

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j DROP
```

On `client`:

```bash
curl -m 5 -I http://10.0.2.10         # -m 5: give up after 5 seconds
sudo nmap -sS -p 22,80 10.0.2.10
```

Expect `curl` to time out (`curl: (28)`), and nmap to show 22 open and 80 **filtered**. On `server`, list the rules and watch the counter on your rule climb with each attempt. Clean up with `sudo iptables -F`.

## Exercise 2: DROP versus REJECT (Verified for A to C)

Part A, REJECT with a TCP reset:

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j REJECT --reject-with tcp-reset
```

From `client`: `time curl -m 10 -I http://10.0.2.10` fails immediately ("Connection refused") and nmap reports 80 **closed**.

Part B, capture on `server` (`sudo tcpdump -ni enp0s3 port 80`) while repeating the `curl`: a SYN arrives and an RST goes straight back.

Part C, swap to DROP (`sudo iptables -F`, then add the DROP rule). `curl` hangs until the timeout, nmap says **filtered**, and tcpdump shows SYNs arriving repeatedly (client retransmits) with nothing sent back.

Part D (Not yet run), plain `-j REJECT` replies with an ICMP "port unreachable" instead of a TCP reset. I expect nmap to label the port `filtered` (it treats ICMP unreachable errors that way for SYN scans) and curl to still fail immediately; capture with `icmp or port 80` to see the ICMP packet. Confirm this in your lab.

| | Client sees | nmap says | tcpdump on server |
|---|---|---|---|
| DROP | Hang, then timeout | filtered | SYNs in, nothing out |
| REJECT (tcp-reset) | Instant "refused" | closed | SYN in, RST out |

Clean up with `sudo iptables -F`.

## Exercise 3: Default-deny policy (Verified)

Add the allow rules first, then set the policy last:

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -L INPUT -v -n --line-numbers
sudo iptables -P INPUT DROP
```

From `client`:

```bash
ping -c 3 10.0.2.10                  # works
ssh user@10.0.2.10                   # works
curl -m 5 -I http://10.0.2.10        # times out
sudo nmap -sS -p 22,80,45678 10.0.2.10     # 22 open; 80 and 45678 filtered
```

A full `-p-` scan shows only port 22 and collapses everything else into `Not shown: 65534 filtered tcp ports`. Services still run (`ss -tlnp` still lists them) but are not reachable. Anything `server` starts itself (such as `apt update`) will also fail, because replies arrive through INPUT and are dropped. Lesson 3 fixes that.

**Recover (order matters):**

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -F
sudo iptables -L INPUT -v -n          # confirm policy ACCEPT, no rules
```

**Gotcha:** finishing the exercise without running the recovery leaves the DROP policy loaded into later lessons. This is the most common cause of "my scan can't find the service". The policy line (for example `policy DROP 262K packets`) shows how much was dropped.

## Exercise 4: Rule order (Verified)

iptables checks rules top to bottom and stops at the first match.

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j DROP
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -L INPUT -v -n --line-numbers
```

`curl -m 5 -I http://10.0.2.10` from `client` times out. Rule 1's counter climbs; rule 2 stays at 0.

Insert the ACCEPT above the DROP, then delete by number:

```bash
sudo iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT      # curl now works
sudo iptables -L INPUT -v -n --line-numbers
sudo iptables -D INPUT 2                                  # delete the rule now numbered 2
```

Rule numbers shift after every insert or delete, so always list with `--line-numbers` immediately before `-D`. Clean up with `sudo iptables -F`.

**Same exercise in nft (Not yet run).** nft starts with nothing, so create a table and chain:

```bash
sudo nft add table inet lab
sudo nft add chain inet lab input '{ type filter hook input priority 0; policy accept; }'
sudo nft add rule inet lab input tcp dport 80 drop        # append, like -A
sudo nft add rule inet lab input tcp dport 80 accept
sudo nft insert rule inet lab input tcp dport 80 accept   # at the top, like -I INPUT 1
sudo nft -a list ruleset                                  # -a shows handles
sudo nft delete rule inet lab input handle 5              # handles do not shift
sudo nft delete table inet lab                            # clean up
```

A packet passes through every table on the hook, and an accept in one table does not override a drop in another, so use one tool at a time per exercise.

## Lesson 2 checkpoint questions and answers

1. *What does nmap report for a port blocked by DROP versus REJECT with a TCP reset, and why?* DROP gives `filtered` because no reply comes back. REJECT with a reset gives `closed` because the server answers like a closed port.
2. *What does tcpdump show for each?* DROP: SYNs arrive (and are retransmitted), nothing goes out. REJECT: a SYN arrives and an RST goes straight back.
3. *Why set the default policy to DROP after adding allow rules?* The policy applies immediately to every packet that matches no rule. With no allow rules yet, that includes your own SSH session. The console is your fallback either way.
4. *Why did the second rule never match, and how do the counters show it?* The DROP was rule 1, matched first, and iptables stops at the first match. Rule 1's packet counter climbs; rule 2's stays at 0.
5. *A service listens on port 9000 but a client cannot connect. Name two places the problem could be.* The firewall (a DROP rule or policy) and the bind address (a service on `127.0.0.1` is local-only). Also check that the client is using the right address and port.
6. *What is the difference between `-A` and `-I`?* `-A` appends to the end of the chain; `-I INPUT n` inserts at position n (default 1). Position decides which rule wins.
7. *Why is deleting by number without listing risky?* Numbers are positions, not permanent labels, so they shift after every change and you may delete the wrong rule (nft handles do not shift).
8. *To block one IP from port 80 but allow everyone else, which rule comes first?* The specific block, because the first match wins. `-A INPUT -p tcp -s 10.0.2.20 --dport 80 -j DROP`, then the general ACCEPT.

# 4. Lesson 3: Stateful filtering

**Concepts:** connection tracking (`conntrack`) remembers each connection so one rule can allow replies to connections you started, without opening ports.

| State | Meaning |
|---|---|
| NEW | First packet of a connection (a TCP SYN, for example) |
| ESTABLISHED | Part of a connection already seen in both directions, including replies |
| RELATED | A new flow tied to an existing connection, such as an ICMP error about it |
| INVALID | Fits no known connection |

**Before you start:** snapshot both VMs; use the `server` console; confirm a clean baseline. On `client`, in its own window, run a simple service for `server` to connect to (your VMs have no internet while the NAT cable is unplugged):

```bash
python3 -m http.server 8000
```

## Exercise 1: See the problem, then fix it (Not yet run)

On `server`, build default-deny:

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP
```

Make `server` start connections to `client`:

```bash
curl -m 5 -I http://10.0.2.20:8000
ping -c 2 -W 2 10.0.2.20
```

Both fail. With `sudo tcpdump -ni enp0s3 port 8000` running, you see the server's SYN go out and the client's SYN-ACK come back repeatedly: the reply reaches the server and INPUT drops it. Add the stateful rule at the top:

```bash
sudo iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Repeat the `curl` and `ping`; both work, and rule 1's counter climbs. Nothing new is allowed in from outside: scan the server from `client` and confirm.

## Exercise 2: Inspect the connection table (Not yet run)

```bash
sudo conntrack -L                         # list tracked connections
sudo conntrack -E                         # live event stream (run during a curl)
cat /proc/sys/net/netfilter/nf_conntrack_count
```

Entries show both directions of one connection plus a countdown timer. In the event stream you see `[NEW]`, `[UPDATE]` and `[DESTROY]` as the TCP state moves through `SYN_SENT`, `SYN_RECV`, `ESTABLISHED`, `TIME_WAIT`.

## Exercise 3 (revised): Why the RELATED state matters (Corrected, Verified)

**Goal:** see that ICMP error replies are classified RELATED, that a default-deny firewall drops them unless you allow RELATED, and what that does to ordinary applications. Nmap is not a good detector for this, and the exercise shows why.

**Setup.** On `server`:

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP
```

On `client`, confirm nothing listens on UDP 5555 (`sudo ss -ulnp | grep 5555` prints nothing). In a second `server` window:

```bash
sudo tcpdump -ni enp0s3 'udp port 5555 or icmp'
```

**Part A: no state rule.**

```bash
sudo iptables -Z                          # zero the counters
sudo nmap -sU -p 5555 10.0.2.20
nc -uzv -w 3 10.0.2.20 5555 && echo "no error received" || echo "error received (port closed)"
sudo iptables -L INPUT -n -v --line-numbers
```

Observed in your lab:

- nmap reports `closed` even though no rule allows the reply. Nmap reads packets off the interface through libpcap, before the INPUT chain, so it sees the ICMP error that iptables then drops (policy line `DROP 1 packets, 56 bytes`).
- `nc` prints `succeeded!` and `no error received`. This is a **false positive**: the port is closed, but the ICMP error was dropped before `nc`'s socket could learn of it.

**Part B: see how the replies are classified.** Rules with no target only count packets:

```bash
sudo iptables -I INPUT 1 -p icmp -m conntrack --ctstate RELATED
sudo iptables -I INPUT 1 -p icmp -m conntrack --ctstate ESTABLISHED
sudo iptables -Z
nc -uzv -w 3 10.0.2.20 5555
sudo iptables -L INPUT -n -v --line-numbers
```

The RELATED counter reads 2 packets and 114 bytes, ESTABLISHED reads 0, and the policy drops the same 2 packets. The errors are RELATED, never ESTABLISHED, so an ESTABLISHED-only rule cannot accept them.

**Part C: allow RELATED and compare.**

```bash
sudo iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -Z
nc -uzv -w 3 10.0.2.20 5555 && echo "no error received" || echo "error received (port closed)"
sudo iptables -L INPUT -n -v --line-numbers
```

Now `nc` prints `error received (port closed)` and the policy line reads `DROP 0 packets`. OpenBSD netcat prints nothing for a failed UDP probe and simply exits with status 1, which is why the `&&`/`||` wrapper helps. Rule 1 accepts all ESTABLISHED and RELATED traffic, so its counter includes other packets (such as an open SSH session), and the counting rules below it show 0 because iptables stops at the first match. To isolate the ICMP packets, put a counting rule **above** the ACCEPT, as in Part B.

**Part D: read the capture.** Your capture showed two UDP probes from `nc` (each `length 1`, a fraction of a millisecond apart) and two ICMP "port unreachable" replies (each `length 37`). The size works out as 8 (ICMP header) + 20 (quoted IP header) + 8 (quoted UDP header) + 1 (payload) = 37, plus the reply's own 20-byte IP header = 57 bytes on the wire, so 2 packets are 114 bytes. Nmap's empty probe gives 36 and 56.

**Clean up:** `sudo iptables -P INPUT ACCEPT` then `sudo iptables -F`.

## Exercise 4: A complete stateful ruleset (Not yet run)

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -P INPUT DROP
```

Loopback is allowed, replies are allowed, invalid packets are dropped, new connections are allowed only to SSH and HTTP, and pings are allowed. Everything else hits the DROP policy. Test: from `client`, `sudo nmap -sS -p 22,80,45678 10.0.2.10` shows 22 and 80 open and 45678 filtered; from `server`, `curl -m 5 -I http://10.0.2.20:8000` and `ping -c 2 10.0.2.20` still work.

## Exercise 5: Rate-limit new SSH connections (Corrected, Not yet run)

**Correction:** my first version inserted the rate-limit rules at positions 5 and 6, which in the Exercise 4 ruleset is *below* the SSH ACCEPT (rule 4), so they would never have matched. The SSH ACCEPT is rule 4, so insert at 4 and 5:

```bash
sudo iptables -L INPUT -n --line-numbers      # confirm the SSH ACCEPT is rule 4
sudo iptables -I INPUT 4 -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --set --name SSH
sudo iptables -I INPUT 5 -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP
sudo iptables -L INPUT -v -n --line-numbers   # both must sit above the "tcp dpt:22 ACCEPT" rule
```

Test from `client`:

```bash
for i in 1 2 3 4 5 6; do nc -zv -w 2 10.0.2.10 22; done
```

I expect the first three attempts to connect and later ones to time out, with the block lifting after about 60 seconds without new attempts, and the counter on the DROP rule climbing. Treat the exact count as a prediction until you run it. Keep the console open if you test with real SSH logins.

**Clean up:**

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -F
sudo iptables -L INPUT -v -n
```

## Lesson 3 checkpoint questions and answers

1. *Why did `curl` from `server` fail in Exercise 1 even though OUTPUT was unrestricted?* The outgoing SYN left through OUTPUT, but the client's SYN-ACK came back through INPUT and hit the DROP policy.
2. *What does the ESTABLISHED,RELATED rule allow, and why can't outsiders start connections with it?* It accepts only packets belonging to connections conntrack already tracks. A new inbound SYN is state NEW, so it does not match.
3. *Revised Exercise 3, why did nmap report `closed` with the reply dropped?* Nmap captures packets at the interface through libpcap before the INPUT chain filters them.
4. *Why did `nc` report success without a RELATED rule, and what does it tell you about UDP results?* The ICMP error was dropped, so the socket never learned of it. A UDP probe with no error is ambiguous (open, or the error was lost), so UDP results are less certain than TCP.
5. *In Part B, which state were the ICMP errors in?* RELATED, never ESTABLISHED, so an ESTABLISHED-only rule cannot accept them.
6. *In Part C, why do the counting rules below rule 1 show 0?* First match wins: rule 1 accepted the packets, so they never reached the rules below it.
7. *Name a real-server situation where dropping RELATED traffic causes a problem.* ICMP errors such as "fragmentation needed" used for path MTU discovery are RELATED. Dropping them can make some connections stall (small packets work, large ones hang), and applications may also wait for timeouts instead of getting quick errors.
8. *Why match SSH and HTTP on `--ctstate NEW` in Exercise 4?* It states the intent explicitly: only connection-opening packets are admitted to those ports, and the rest of each session rides on the ESTABLISHED rule.
9. *Why must the rate-limit rules go above the SSH ACCEPT?* First match wins. If the ACCEPT came first, SSH packets would be accepted before the `recent` rules ever saw them.

# 5. Command reference

## nmap

| Switch | Meaning |
|---|---|
| `-sS` | SYN scan (needs sudo; default as root) |
| `-sT` | Connect scan (default without root; completes the handshake) |
| `-sU` | UDP scan (needs sudo; slow; often open or filtered) |
| `-sV` | Version detection on open ports |
| `-p 22,80` | Scan listed ports; ranges allowed (`-p 1-1024`) |
| `-p-` | All 65,535 ports |
| (no `-p`) | Top 1000 common ports |
| `-sn` | Ping sweep, no port scan (`nmap -sn 10.0.2.0/24`) |
| `-Pn` | Skip host discovery (needed when ping is blocked) |
| `-v` | Verbose |
| `--reason` | Show why each port got its state |

Only scan machines you own or may test.

## ss

`sudo ss -tlnp` is four flags: `-t` TCP only, `-l` listening only, `-n` numeric, `-p` show the owning process (needs sudo).

| Flag | Meaning |
|---|---|
| `-u` | UDP sockets |
| `-a` | All sockets, including active connections |
| `-4` / `-6` | IPv4 or IPv6 only |
| `-s` | Summary counts |
| `-i` | TCP internals (RTT, congestion window) |

Read **Local Address** first: `127.0.0.x` is local only; `0.0.0.0` or `[::]` is all interfaces. `ss` replaces `netstat` (same flags for most tasks; routes moved to `ip route`).

## nc (OpenBSD netcat)

| Switch | Meaning |
|---|---|
| `-l` | Listen |
| `-k` | Keep listening after a client disconnects (with `-l`) |
| `-u` | UDP |
| `-z` | Scan mode: connect or probe, send no data |
| `-v` | Verbose: prints a line on success |
| `-w 3` | Timeout in seconds |

Listener syntax: `nc -l -k 45678` (the port is an argument; `-p` sets a source port). `nc -uzv` prints `succeeded!` (exit 0) when no ICMP error arrives, and prints nothing (exit 1) when the port is closed. A background `nc ... &` can stop with `Stopped (tty input)`; run it in its own window instead.

## curl

| Switch | Meaning |
|---|---|
| `-I` | HEAD request: headers only (uppercase i) |
| `-i` | GET, but print headers too |
| `-m 5` | Give up after 5 seconds total (DROP gives `curl: (28)`; REJECT gives `(7)` instantly) |
| `--connect-timeout 5` | Limit only the connection phase |

## tcpdump

| Switch | Meaning |
|---|---|
| `-n` | No name lookups |
| `-i enp0s3` | Interface (check with `ip -br a`) |
| `-S` | Absolute sequence numbers |
| `-v`, `-vv` | More detail |
| `-X` | Packet contents in hex and ASCII |
| `-c 20` | Stop after 20 packets |
| `-w file.pcap` | Write to a file |

## systemctl

`sudo systemctl enable --now nginx ssh` enables at boot and starts now. Also: `status`, `start`, `stop`, `restart`, `disable`, `disable --now`. Enabled (starts at boot) and active (running now) are independent.

## iptables

| Option | Meaning |
|---|---|
| `-A CHAIN` | Append a rule |
| `-I CHAIN [n]` | Insert at position n (default 1) |
| `-D CHAIN n` | Delete rule number n (or by matching the rule) |
| `-F` | Flush rules (policy unchanged) |
| `-P CHAIN POLICY` | Set default policy |
| `-L -v -n --line-numbers` | List with counters, numeric, with rule numbers |
| `-Z` | Zero counters |
| `-p tcp --dport 80` | Match protocol and destination port |
| `-s IP` / `-i IF` | Match source address / input interface |
| `-m conntrack --ctstate S` | Match connection state |
| `-j ACCEPT / DROP / REJECT` | Action. `REJECT --reject-with tcp-reset` answers with a reset |
| (no `-j`) | Count only, then continue |

## nft

| Task | Command |
|---|---|
| List everything | `sudo nft list ruleset` (add `-a` for handles) |
| Create table | `sudo nft add table inet lab` |
| Create base chain | `sudo nft add chain inet lab input '{ type filter hook input priority 0; policy accept; }'` |
| Append / insert | `nft add rule ...` / `nft insert rule ...` |
| Delete by handle | `nft delete rule inet lab input handle N` |
| Delete table | `nft delete table inet lab` |

# 6. Troubleshooting notes from the lab

| Symptom | Likely cause and check |
|---|---|
| nmap cannot find a running service | Leftover default-deny policy. Run `sudo iptables -L INPUT -v -n` first. Then check `ss -tlnp` for the listener and its bind address |
| Service listed in `ss` but unreachable | Firewall drop, or bound to a loopback address |
| Only port 22 appears in a full scan | Policy DROP with an SSH allow rule: other ports are counted in the "Not shown ... filtered" line |
| Port 53 not visible from `client` | `systemd-resolved` listens on loopback only (`127.0.0.53`, `127.0.0.54`). Do not disable it |
| `netplan apply`: cannot find matching interface | Wrong interface name or stale `match:` block (section 1.4) |
| Clone fails: "could not create directory" | Set the clone Path explicitly; check drive letter, free space, NTFS or exFAT, leftover half-created folder, Controlled Folder Access, USB power saving; test by cloning to the internal drive |
| VM very slow, or "VT-x not available" | Virtualization off in BIOS, or Hyper-V competing |
| No internet in VM during install or update | Adapter 1 must be NAT (and Cable Connected ticked) |
| Locked out by a firewall rule | Use the console and `sudo reboot` (rules are not persistent) |
| `nc` background job stops | `Stopped (tty input)`: run it in its own window without `&` |

# 7. Revision log and open items

**Corrections made during the lab**

| # | What was wrong | Correction |
|---|---|---|
| 1 | Lesson 1 Ex 1 said a default nmap scan misses port 9000 | 9000 is in nmap's top 1000. Use 45678 |
| 2 | Lesson 3 Ex 3 said nmap would show `open or filtered` without a RELATED rule | Nmap reads packets before iptables and reports `closed` either way. The exercise now uses `nc` exit status, counters and a capture |
| 3 | Said `nc -uzv` would print "Connection refused" | OpenBSD netcat prints nothing on failure and exits 1; check the exit status as in Lesson 3 Ex 3 Part A |
| 4 | Predicted the ICMP counter would read 1 packet, 56 bytes | `nc` sends two 1-byte probes, so 2 packets, 57 bytes each (114). Nmap's empty probe gives 56 |
| 5 | Described `ss -tlnp` as "five flags" | It is four: `-t -l -n -p` |
| 6 | Used `nc -l -k -p 9000` | The port is an argument: `nc -l -k 9000` |
| 7 | Lesson 3 Ex 5 inserted rate-limit rules at positions 5 and 6 | They must go above the SSH ACCEPT (rule 4): insert at 4 and 5 |
| 8 | Said Lesson 3 prerequisites assumed no internet | A working NAT Adapter 2 gives the VM a route out; untick Cable Connected during exercises |

**Not yet run in your lab:** Lesson 2 Ex 2 Part D and the nft version of Ex 4; Lesson 3 Exercises 1, 2, 4 and 5.

**Planned additions:** Lesson 4 (nftables: tables, sets, saving rulesets), Lesson 5 (NAT and routing with the `gateway` VM), Lesson 6 (logging and debugging), Lesson 7 (capstone with automated tests), and optional topics (cloud firewalls, Kubernetes NetworkPolicies, WAFs, IPv6). Corrections found while working through them will be added to the log above.
