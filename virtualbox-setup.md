# Virtual Box Setup

Here’s the full setup for VirtualBox on Windows 11. The key idea is to build one VM with internet access, install your tools, snapshot it, then clone it and rewire the network for each lab role.

## Step 1: Check prerequisites

- **Hardware virtualization must be on**. Open Task Manager → Performance → CPU and look for “Virtualization: Enabled.” If it says Disabled, reboot into your BIOS/UEFI and enable Intel VT-x or AMD-V (sometimes labeled SVM Mode).

- **Disk space:** plan for about 40 GB free (three VMs at roughly 10 GB each, plus the ISO and snapshots).

- **RAM:** each VM needs about 1 GB, so 8 GB of host RAM is workable and 16 GB is comfortable.
  
- **Hyper-V and WSL2 conflicts:** Windows 11 often has Hyper-V features enabled (WSL2, Docker Desktop, and Memory Integrity can all turn them on). Current VirtualBox versions can run alongside Hyper-V, but VMs may run noticeably slower. If performance is poor, you can disable “Virtual Machine Platform” and “Windows Hypervisor Platform” under Turn Windows features on or off and reboot, but that will break WSL2 until you re-enable them.

## Step 2: Install VirtualBox
- Download the Windows installer from virtualbox.org (Downloads → “Windows hosts”).

- Run it as administrator and accept the defaults. The installer will briefly disconnect your network adapters while it installs the networking drivers, which is normal.

- Optionally, download the Extension Pack from the same page. You don’t need it for this lab.

## Step 3: Download Ubuntu Server

Get the Ubuntu Server 24.04 LTS (or 22.04 LTS) ISO from ubuntu.com/download/server. The Server edition has no desktop and uses far fewer resources, which suits this lab.

## Step 4: Create the base VM

- In VirtualBox, click **New**.

- Name it ubuntu-base, select your ISO, and check **Skip Unattended Installation** so you control the install.

- Set **1024 to 2048 MB** of RAM and **1 to 2 CPUs**.
  
- Create a **10 GB** virtual disk (dynamically allocated).
  
- Before starting, open **Settings → Network**. Leave Adapter 1 as **NAT** for now so the VM can reach the internet during install.
*Complete the above step after creating the VM but before starting it up*

- Start the VM and walk through the Ubuntu installer. Accept the defaults, create a user and password, and check **Install OpenSSH** server when offered. Skip the optional snaps.

- When it finishes, remove the ISO if prompted and reboot.

## Step 5: Install your tools

- Log in at the VM console and run:

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y nftables iptables ufw tcpdump nmap netcat-openbsd curl conntrack nginx
```
*Another option to cut and paste the commands above is to do SSH to the VM via port forwarding.  See instructions below*

>SSH lets you use Windows Terminal or PowerShell, where copy and paste work normally. It also gives you scrollback and a better terminal than the VM console.
>
>**While Adapter 1 is NAT (during setup and base-VM work):**
>
>- In VirtualBox, open the VM’s **Settings → Network → Adapter 1 → Advanced → Port Forwarding**.
>- Add a rule: Protocol **TCP**, Host IP **127.0.0.1**, Host Port **2222**, Guest IP blank, Guest Port **22**.
>- Make sure OpenSSH server is installed in the VM (sudo apt install openssh-server), then from Windows run:
>
>```
>ssh -p 2222 youruser@127.0.0.1
>```
>
>**Once your VMs move to Internal Network**: port forwarding no longer applies, so add a second (or third) adapter set to **Host-Only Adapter** (Settings → Network → Adapter 2). >Give it an address in the host-only subnet (usually 192.168.56.0/24) and SSH to that address instead


- Shut down with sudo poweroff, then take a snapshot: Machine Tools → Snapshots → Take, named base-tools.

## Step 6: Clone into lab VMs

- Right-click ubuntu-base → **Clone**.

- Name it **client**. Set MAC Address Policy to **Generate new MAC addresses for all network adapters**. Choose **Full clone**.
  
- Repeat for server, and later for gateway (when you reach Lesson 5).

## Step 7: Configure the lab networks

Use VirtualBox Internal Network adapters, which are isolated from your host and home network. For Lessons 1 to 4, you only need client and server on the same segment:

| VM | Adapter 1 | Adapter 2 | Adapter 3 |
|----|-----------|-----------|-----------|
| client | Internal Network labnet | optional: NAT (for updates) | Host-Only Adapter (for SSH access) |
| server | Internal Network labnet | optional: NAT (for updates) | Host-Only Adapter (for SSH access) |

For Lessons 5 and 7, rename the segments to two networks:

| VM | Adapter 1 | Adapter 2 | Adapter 3 | Adapter 4 |
|----|-----------|-----------|-----------|-----------|
| client | Internal netA | | Host-Only Adapter (for SSH access) | |		
| gateway | Internal netA	| Internal netB	| Host-Only Adapter (for SSH access) | NAT | 
| server | Internal netB | | Host-Only Adapter (for SSH access) | |		

Set each in **Settings → Network → Adapter N → Enable → Attached to: Internal Network**, then type or pick the name.

*To make it easier to connect to the VMs from the Windows 11 host machine, enable Adapter 3 to "Host-Only Adapter" for each VM. We will set the static for the Host-Only LAN in the section below*

Two things to know:

- **Adapter order matters**. Inside Ubuntu the interfaces will appear in order (commonly enp0s3, enp0s8, enp0s9). Confirm with ip -br a before editing netplan.
- **An internal-only VM has no internet**. Do any apt install work before switching to Internal Network, or keep a second NAT adapter for updates (just remember it adds a default route that can confuse your routing tests in Lesson 5).

## Step 8: Assign static IPs

LabNet LAN
- Server VM - 10.0.2.10
- Client VM - 10.0.2.20

Host-Only LAN
- Server VM - 192.168.56.10
- Client VM - 192.168.56.20

Edit netplan on each VM (sudo nano /etc/netplan/50-cloud-init.yaml or similar), using the interface name you saw in ip -br a. For server on labnet:


### Server VM
```
yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.0.2.10/24]
    enp0s9:
      addresses: [192.168.56.10/24]
```

### Client VM

```
yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.0.2.20/24]
    enp0s9:
      addresses: [192.168.56.20/24]
```
*After enabling **Adapter 3** as **Host-Only Adapter** in the two VMs, we'll configure it with static IP from the Host-Only LAN - 192.168.56.x which enables SSH from the host machine*

Apply with sudo netplan apply, then confirm connectivity with ping 10.0.2.10 from the client.

*When applying the new netplan with the static ip address, there's an error because of the existence of another netplan file.  To fix this, run the following commands*

```
  sudo mkdir -p /root/netplan-backup
  sudo mv /etc/netplan/00-installer-config.yaml /root/netplan-backup/   # use your actual filename
  sudo netplan apply
```

### Step 9: Create SSH profiles on Tabby

For each VM:
- On the **Tabby** application, go to **Settings**
- Select **Profiles & connections** from the left column
- Select **+ New -> New Profile -> SSH connection**
- For Name, enter either "server" or "client"
- For Host, enter the respective IP address - 192.168.58.10 for server or 192.168.58.20 for client
- For Username, enter the username you selected
- For Authentication Method, select **Password**. There's an option to save password under **Set Password**
- Select **Save** at the bottom of the window

