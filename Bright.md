# Bright Cluster Manager (BCM) – Installation and Configuration Guide
This work credit Kiran Gospal 13OCT25


## Environment Overview
| Component | Hostname | Role | Network | IP Address |
|------------|-----------|------|----------|-------------|
| Head Node | `server147` | Head Node / Cluster Manager | internalnet | `172.28.103.147` |
| Compute Node 1 | `server145` | Worker / Compute | internalnet | `172.28.103.146` |
| Compute Node 2 | `server146` | Worker / Compute | internalnet | `172.28.103.148` |
| Management | internalnet | Cluster communication, PXE, DHCP, provisioning | 172.28.103.0/24 |
| External | externalnet | Admin / Internet access | 172.28.2.0/24 |

> Head node manages PXE boot, DHCP, provisioning, and monitoring over the **internalnet**.

---

## 1. Head Node Setup

1. Install Bright Cluster Manager from the provided ISO image (Ubuntu base).
   Intallation notes available at https://docs.nvidia.com/dgx-basepod/deployment-guide-dgx-basepod/latest/bcm-deploy.html
2. Activate the cluster license :
   ```bash
   request-license
   ```
3. Configure network interfaces:
   - **eno1 / externalnet** → `172.28.2.147`
   - **eno2 / internalnet** → `172.28.103.147`
4. Verify DHCP and TFTP are active:
   ```bash
   systemctl status cm-dhcpd
   ls /var/lib/tftpboot/
   ```
5. Confirm `cmd` service is running:
   ```bash
   systemctl status cmd
   ```
6. Run the Bright console:
   ```bash
   cmsh
   ```
   and ensure the cluster partition and internal network (`internalnet`) are visible.

---

## 2. Adding Compute Nodes

### 2.1. Define and Configure the Node
Example: Adding `server146` (MAC `4C:D9:8F:3F:A2:06`)

```bash
cmsh
device
add server146
set server146 category default
set server146 interfaces 1 BOOTIF
set server146 macaddress 1 4C:D9:8F:3F:A2:06
set server146 ip 1 172.28.103.148
commit
```

> `BOOTIF` is the PXE NIC for provisioning (usually `eno2`).

---

### 2.2. Fix Interface Naming and Bootability

Rename interface and set the correct MAC and boot flags:

```bash
device
use server146
interfaces
use BOOTIF
set networkdevicename eno2
set mac 4C:D9:8F:3F:A2:06
set bootable yes
commit
```

Set provisioning interface:
```bash
up
up
set provisioninginterface eno2
set installbootrecord yes
set nextinstallmode FULL
set blockdevicesclearedonnextboot sda
set pxelabel install
commit
```

> `installbootrecord yes` ensures the bootloader is written to disk for local boot.  
> `blockdevicesclearedonnextboot sda` wipes the disk before reinstall.

---

### 2.3. PXE Boot the Node

1. In **iDRAC**, enable PXE boot on the NIC with the matching MAC.  
2. Power on the node — it will:
   - Request an IP from DHCP (`172.28.103.x`)
   - Boot from network
   - Perform a full OS installation  
   - Write GRUB to local disk

---

### 2.4. Verify Successful Installation

From the head node:
```bash
cmsh -c "device; list"
```

Expected:
```
server145  [UP], health check OK
server146  [UP], health check OK
```

To check node info:
```bash
cmsh -c "device; use server146; show"
```

> Once installed, the console on the node displays:
> ```
> Base Command Manager trunk
> Head Node: server147
> Hostname: server146
> Slurm node daemon started
> ```
> `/etc/os-release` shows `Ubuntu 24.04.1 LTS` — this is **normal** (Bright uses Ubuntu as its base image).

---

## 3. Post-Installation Configuration

### 3.1. Prevent Reinstall on Reboot
After the first successful install:
```bash
cmsh -c "device; use server146; set pxelabel main; commit"
```
Then disable PXE boot in BIOS or move **Hard Disk** to top of boot order.

---

### 3.2. Verify Node Health
```bash
cmsh
mon
show device server146
```
Check:
- `Node state : running`
- `Health : OK`

---

### 3.3. Check Services
Ensure Slurm, cmd, and monitoring agents are active:
```bash
systemctl status slurmd
systemctl status cmd
systemctl status cmdaemon
```

---

## 4. Additional notes

| Action | Command |
|--------|----------|
| Check all devices | `cmsh -c "device; list"` |
| View detailed node info | `cmsh -c "device; use <node>; show"` |
| Reinstall a node | `cmsh -c "device; use <node>; set nextinstallmode FULL; set pxelabel install; commit"` |
| Force full disk wipe | `cmsh -c "device; use <node>; set blockdevicesclearedonnextboot sda; commit"` |
| Monitor logs | `journalctl -u cmd -n 50` |
| Tail install log (while installing) | `tail -f /var/log/syslog | grep installlog` |

---

## 5. Expected End State

**Cluster Components**
- Head node: `server147` (BCM + Slurm Controller)
- Compute nodes: `server145`, `server146` (Slurm clients)
- Network segregation: `externalnet` (admin), `internalnet` (provisioning)
- PXE and DHCP functioning
- Nodes boot from local disk post-installation
- Health checks reporting OK

 **OS**
- All nodes based on Ubuntu 24.04 LTS
- Bright services layered over Ubuntu (`cmd`, `bcmdaemon`, `slurmd`, etc.)

---

## 6. Summary

| Step | Description | Result |
|------|--------------|--------|
| 1 | Install and configure BCM on head node | Head node ready |
| 2 | Add compute nodes via `cmsh` | Nodes defined |
| 3 | Configure PXE + provisioning | Nodes boot and install |
| 4 | Verify and set PXE label to `main` | Nodes boot from disk |
| 5 | Confirm services and monitoring | Cluster operational |

---

 **Cluster is now fully functional under Bright Cluster Manager on Ubuntu 24.04**  
PXE provisioning, Slurm integration, and internal network communication are verified.
