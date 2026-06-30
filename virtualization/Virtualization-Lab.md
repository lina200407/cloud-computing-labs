# Lab 01 — Introduction to Virtualization with VirtualBox

> **Course:** Cloud Computing | **Date:** March 2025  
> **Goal:** Set up two Lubuntu VMs, configure multiple network modes, and establish SSH connectivity between them.

---

## What this lab covers

- Installing VirtualBox and verifying hardware virtualization support
- Creating a VM from an ISO and configuring its resources
- Installing and testing an SSH server inside a VM
- Cloning a VM via OVA export/import
- Understanding NAT vs Internal vs Bridged networking
- SSHing from host → VM → VM (jump host pattern)

---

## Step 1 — Installing VirtualBox and verifying CPU virtualization

VirtualBox requires hardware-assisted virtualization (Intel VT-x or AMD-V) to be enabled in the BIOS/UEFI. Without it, 64-bit VMs won't run.

**How to verify it's enabled on Linux:**

```bash
grep -E --color 'vmx|svm' /proc/cpuinfo
```

- `vmx` → Intel VT-x is active
- `svm` → AMD-V is active

If the output is empty, you need to enable virtualization in your BIOS settings and reboot.

You can also check with:

```bash
lscpu | grep Virtualization
```

Expected output (Intel):
```
Virtualization: VT-x
```

---

## Step 2 — Creating the Lubuntu-01 VM

Download the Lubuntu ISO from [lubuntu.me](https://lubuntu.me), then create a new VM in VirtualBox.

**Resources allocated to the VM:**

| Resource | Value | Why |
|---|---|---|
| RAM | 1024 MB | Lubuntu is lightweight; 1 GB is sufficient |
| CPUs | 1 | Single core is fine for a lab VM |
| Disk | 10 GB (dynamically allocated) | Saves host disk space |
| Network | NAT (initial) | Lets the VM reach the internet through the host |

During installation:
- Hostname: `Lubuntu-01`
- Username: your family name (as required by the lab)

---

## Step 3 — Installing and testing SSH

NAT mode gives the VM internet access, so we can install packages.

**Install the OpenSSH server:**

```bash
sudo apt update && sudo apt install openssh-server
```

**Check that the SSH service is running:**

```bash
systemctl status ssh
```

You're looking for `Active: active (running)` in green. The output looks like:

```
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled)
     Active: active (running) since ...
```

**Test it locally (loopback test):**

```bash
ssh localhost
```

If it connects and shows a shell prompt (after accepting the host key), the SSH server is working correctly. This confirms both that the daemon is listening on port 22 and that it accepts connections.

---

## Step 4 — Cloning the VM via OVA export/import

Instead of installing Lubuntu a second time, we export the configured VM and import it as a clone. This is exactly how VM templates work in cloud environments.

**Export Lubuntu-01 as an OVA:**

In VirtualBox: `File → Export Appliance → select Lubuntu-01 → save as .ova`

An OVA (Open Virtualization Archive) bundles the disk image and VM configuration into a single portable file.

**Import the OVA to create Lubuntu-02:**

`File → Import Appliance → select the .ova file → rename to Lubuntu-02`

**Rename the hostname from inside the VM:**

The cloned VM still thinks its name is `Lubuntu-01`. Fix it:

```bash
sudo hostnamectl set-hostname Lubuntu-02
```

Then edit `/etc/hosts` to update the old hostname reference:

```bash
sudo nano /etc/hosts
```

Replace any line containing `Lubuntu-01` with `Lubuntu-02`. Apply without rebooting:

```bash
exec bash
```

Or just reboot the VM. Verify:

```bash
hostname
```

Expected output: `Lubuntu-02`

---

## Step 5 — Configuring the network interfaces

This step sets up three different network modes across the two VMs:

| VM | Interface | Mode | Purpose |
|---|---|---|---|
| Lubuntu-01 | eth0 (enp0s3) | Internal | Private link to Lubuntu-02 |
| Lubuntu-01 | eth1 (enp0s8) | Bridged | Reachable from the host machine |
| Lubuntu-02 | eth0 (enp0s3) | Internal | Private link to Lubuntu-01 |

**Why Internal mode?**  
Internal networking creates a private virtual switch between VMs only — no host access, no internet. It simulates an isolated LAN.

**Why Bridged mode on Lubuntu-01?**  
Bridged connects the VM directly to your physical network, getting its own IP from your router's DHCP. This makes it reachable from the host just like any other machine on the LAN.

After changing network settings on a running VM, force the network manager to re-read the config:

```bash
sudo systemctl restart NetworkManager
```

**Check IP addresses:**

```bash
ip a
```

On **Lubuntu-01** you'll see two interfaces with IPs — one in your LAN range (bridged, e.g. `192.168.1.x`) and one in the internal network range (e.g. `10.0.2.x` or link-local `169.254.x.x` if no DHCP is configured on the internal network).

On **Lubuntu-02** you'll only see the internal network interface (no internet access).

---

## Step 6 — Establishing SSH connectivity

### From the host → Lubuntu-01 (via bridged interface)

First, verify host can reach the VM:

```bash
ping <Lubuntu-01-bridged-IP>
```

Example output:
```
PING 192.168.1.42 (192.168.1.42): 56 data bytes
64 bytes from 192.168.1.42: icmp_seq=0 ttl=64 time=0.512 ms
```

Then SSH in:

```bash
ssh <your-username>@<Lubuntu-01-bridged-IP>
```

**Confirm you're inside Lubuntu-01:**

```bash
hostname
```

Output: `Lubuntu-01` — you're in.

Or check both hostname and IP at once:

```bash
hostname && ip a | grep inet
```

### From Lubuntu-01 → Lubuntu-02 (via internal network)

While SSH'd into Lubuntu-01, SSH further into Lubuntu-02 using the internal network IP:

```bash
ssh <your-username>@<Lubuntu-02-internal-IP>
```

**Confirm you're inside Lubuntu-02:**

```bash
hostname
```

Output: `Lubuntu-02`

This demonstrates the **jump host pattern** — Lubuntu-01 acts as a gateway into the isolated internal network where Lubuntu-02 lives. In real infrastructure, this is called a bastion host or jump server.

---

## Key concepts summary

| Concept | What it means in practice |
|---|---|
| NAT | VM gets internet via host, but host can't initiate connections to VM |
| Internal | VMs on the same internal network can talk to each other; no outside access |
| Bridged | VM appears as a real device on your LAN; host and other machines can reach it |
| OVA | Portable VM package — the basis for cloud VM templates and golden images |
| Jump host | A VM that bridges two networks, used to reach machines with no direct external access |
