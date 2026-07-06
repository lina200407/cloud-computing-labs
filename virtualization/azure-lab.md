# Lab 02 — Virtualization on the Cloud (Microsoft Azure)

> **Course:** Cloud Computing | **Date:** April 2025  
> **Goal:** Deploy a Windows Server VM on Azure, install IIS, and host a website accessible over the internet.

---

## What this lab is about

This was my first time actually deploying something on a real cloud platform. The goal was straightforward: spin up a Windows Server VM on Microsoft Azure, install a web server on it, and host a website that anyone on the internet can visit. Simple in theory — but there are a lot of moving parts, and this writeup explains what each one is and why it matters.

---

## Concepts worth knowing before you start

### Resource Group
Think of it as a folder for all the things Azure creates when you make a VM — the disk, the network interface, the public IP, the firewall rules, everything. The nice thing is if you want to clean up after a lab, you just delete the resource group and it takes everything with it.

### Region
This is the physical location of the datacenter where your VM runs. We used **South Africa North** (which is in Johannesburg). In real projects, you'd pick a region close to your users for lower latency, or a specific region for legal reasons — some countries require data to stay within their borders.

### Availability Zone
Within a single region there are multiple physically separate datacenters, each with its own power and cooling. If you spread your VMs across zones, one datacenter going down won't take your whole app with it. For a lab we don't bother, but in production this matters.

### Gen2 VM
When you see "x64 Gen2" on a VM image, it means the VM uses **UEFI** firmware instead of the old legacy BIOS. Gen2 VMs boot faster, support larger disks, and unlock modern security features like Secure Boot and vTPM. Most current Azure images are Gen2 only — Gen1 is being phased out.

---

## Step 1 — Setting up the Resource Group and VM

In the Azure Portal: `Resource groups → Create`

Name it something meaningful (e.g. `cloud-lab-rg`) and set the region to **South Africa North**.

Then create the VM: `Virtual Machines → Create → Azure Virtual Machine`

Here's what I configured and why:

| Parameter | Value | Why |
|---|---|---|
| Image | Windows Server 2022 Datacenter: Azure Edition - x64 Gen2 | Modern, UEFI-based Windows Server with Azure-specific optimizations |
| VM Size | B2as v2 | 2 vCPUs, 8 GB RAM — cheap burstable instance, perfect for a lab |
| Storage | Standard SSD | Better than HDD, cheaper than Premium SSD, fine for low-traffic workloads |
| Region | South Africa North | Required by the lab |

A note on the **B-series (burstable)** size: these VMs earn CPU credits when they're idle and spend them when they need to burst. For a lab web server that's mostly sitting around, this is ideal — you're not paying for full CPU performance you'll never use.

Click **Review + Create** → **Create** and wait about 2 minutes for deployment.

---

## Step 2 — Connecting to the VM via RDP

Once the VM is deployed, click **Go to resource**, then:

`Connect → RDP → Download RDP file`

The `.rdp` file already has the VM's public IP and port 3389 filled in. Just open it:
- **Windows:** built-in Remote Desktop Connection
- **Linux:** `xfreerdp` or Remmina  
- **macOS:** Microsoft Remote Desktop from the App Store

Log in with the admin username and password you set during creation.

If you're wondering why we're using RDP instead of SSH — Windows Server doesn't ship with an SSH server running by default. RDP is the native way to get a full graphical desktop session on a Windows machine remotely. Port 3389 is to Windows what port 22 is to Linux.

---

## Step 3 — Installing IIS

IIS (Internet Information Services) is Microsoft's built-in web server. It doesn't come installed — you add it as a "role" through Server Manager.

Once inside the VM:

1. **Server Manager** opens automatically on login
2. Click **Add Roles and Features**
3. Keep clicking Next until you reach **Server Roles**
4. Check **Web Server (IIS)** — accept any extra features it wants to add
5. Click through to **Install**
6. When it finishes, hit **Refresh** in Server Manager

To verify it worked, open a browser inside the VM and go to `http://localhost`. You should see the default IIS welcome page (blue background, IIS logo). That confirms the web server is up and listening on port 80.

For context: if this were a Linux VM, you'd install Nginx or Apache via `apt`. IIS is the Windows equivalent — it integrates with NTFS permissions, Windows authentication, and Active Directory in ways that third-party servers on Windows don't.

---

## Step 4 — Hosting a Website

### Set up the site in IIS Manager

`Server Manager → Tools → IIS Manager`

1. In the left panel: expand your server → **Sites**
2. **Stop** the Default Web Site (right-click → Stop) — it's already using port 80 and would conflict
3. Copy your website folder into the VM (e.g. `C:\inetpub\mysite\`)

### Create the new site

Right-click **Sites → Add Website**:

| Field | Value |
|---|---|
| Site name | anything you like |
| Physical path | the folder you copied your site into |
| Port | 80 |

### Fix folder permissions

Right-click your website folder → **Properties → Security → Edit → Add**

Add the user `Everyone` with **Read** permission. IIS's worker process runs as a restricted account and won't be able to serve your files unless the folder grants read access. This is a step that's easy to forget and causes a confusing 403 error.

### Open port 80 in Azure — the step most people miss

Azure puts a firewall (called an NSG — Network Security Group) in front of your VM. By default it blocks everything except RDP. You have to explicitly allow HTTP traffic:

`VM → Networking → Add inbound port rule`

| Field | Value |
|---|---|
| Destination port | 80 |
| Protocol | TCP |
| Action | Allow |
| Priority | 100 |

Without this step, your website will work fine from inside the VM but be completely unreachable from outside. The NSG is Azure's version of a security group — a stateful firewall that controls what traffic can reach your VM.

### Visit your site

From your own browser:

```
http://<your-VM-public-IP>/
```

The public IP is on the VM's overview page in the Azure Portal.

---

## Side quest answers

### What does Gen2 mean?

It's about firmware. Gen2 = UEFI, Gen1 = legacy BIOS. UEFI supports larger disks, faster boot, Secure Boot, and vTPM. Most new Azure images dropped Gen1 support entirely, so if you try to create an old-style Gen1 VM with a modern image, it simply won't work.

### Azure terms decoded

| Term | Plain English |
|---|---|
| Region | Which city's datacenter your stuff runs in |
| Availability Zone | Which building within that city — matters for redundancy |
| VM Size | How many CPUs, how much RAM, how fast the network is |
| Image | The pre-built OS snapshot your VM starts from |
| NSG | A firewall that sits in front of your VM in the cloud |
| Public IP | The internet-facing address of your VM |
| Managed Disk | Your VM's hard drive, but Azure handles the physical storage for you |

### Can you use a hostname instead of an IP?

Yes. Azure can assign a DNS name to your VM's public IP:

`VM → Public IP address → Configuration → DNS name label`

Set a label, save, and your site becomes:

```
http://<your-label>.southafricanorth.cloudapp.azure.com/
```

This is more reliable than using the IP directly — if you restart the VM, the IP might change (unless you set it to Static), but the DNS name stays the same.

### Checking your Azure credit

`Azure Portal → search "Cost Management" → Cost Management + Billing`

- **Overview** — see your remaining student credit and what you've spent so far
- **Cost analysis** — break down spend by resource or time period
- **Budgets** — set an alert so Azure emails you before you accidentally burn through your credit

You can also get to it via: `Subscriptions → your subscription → Overview`

---

## What I took away from this

The thing that clicked for me doing this lab was realizing that "the cloud" is just someone else's computer — but with an absurdly good API around it. Everything I did here (picking hardware specs, installing software, configuring a firewall, hosting a website) is the exact same workflow as setting up a physical server. The difference is it took 10 minutes instead of a day, I paid for exactly what I used, and I deleted everything when I was done. That's the IaaS model in a nutshell.
