# 01 - Hypervisor and virtual networks

## Environment
- Host: Intel Core i9-13900K, 64 GB DDR5, NVIDIA RTX 3070, Windows 11 Pro (build 26100)
- Hypervisor: VMware Workstation Pro 25H2 (25.0.0.24995812), free personal-use license
- Backend: Windows Hypervisor Platform (see "Problems encountered")

## Storage layout

| Disk | Device | Type | Role | Reason |
|---|---|---|---|---|
| 3 | Samsung 980 PRO 2 TB | NVMe SSD | `C:\VMs\detection`, `C:\VMs\malware` (running VMs) | VM disk I/O is random and heavy, so the fastest disk holds the running VMs. |
| 1 | WD 8 TB | HDD | `D:\Lab\iso`, `D:\Lab\snapshots-archive`, `D:\Lab\sample-vault` | Installers, archived snapshots and malware samples are read rarely, so a slower disk is fine. |
| 2, 0 | WD 1 TB, Seagate 2 TB | HDD | Not used by the lab | Personal data. |

Rule: fast disk for running VMs, large disk for everything else. The sample vault is on a different physical disk from the running VMs, so samples never sit next to active guests. Samples stay as password-protected zips and are never extracted on the host.

## Network design

| Network | Type | Subnet | DHCP | Host adapter | Purpose |
|---|---|---|---|---|---|
| VMnet1 | Host-only | 10.10.100.0/24 | Off (static IPs) | Yes (10.10.100.1) | Detection lab: DC, SIEM, victims, attacker |
| VMnet2 | Host-only | 192.168.200.0/24 | Off | **No** | Malware analysis, fully isolated |
| VMnet8 | NAT | 192.168.138.0/24 (VMware default) | On (default) | Yes | Updates and installs only |


Why VMnet2 has no host adapter:  A host adapter gives the host an IP address on the network, which is a path from the guest to the host's network stack. Removing it means malware in the VM has no interface to reach the host through, and turning off DHCP means no VM can join the network without a deliberate static address.

## Verification

- `docs/img/01-network-editor.png`: Virtual Network Editor table. Shows VMnet1 and VMnet2 with DHCP disabled and VMnet2 with no host connection. 
- `docs/img/02-ipconfig.png`: `ipconfig` output. Shows a VMnet1 adapter, a VMnet8 adapter, and **no VMnet2 adapter**, which proves the host has no interface on the malware network.
- `docs/img/03-preferences.png`: Workspace preferences. Shows the default VM location set to `C:\VMs\detection`, and shared folders disabled by default.

![Virtual Network Editor](img/01-network-editor.png)
![ipconfig output](img/02-ipconfig.png)
![VMware preferences](img/03-preferences.png)

## Problems encountered

### 1. Virtualization-based security (VBS) kept running

**Symptom:** `msinfo32` reported *Virtualization-based security: Running* and *A hypervisor has been detected*. VMware Workstation runs slower when Windows keeps its own hypervisor active.

**What I checked (evidence):**
- `hypervisorlaunchtype` was `Off` after the first `bcdedit` change.
- `Win32_DeviceGuard`: `VirtualizationBasedSecurityStatus = 2` (running), with no security services configured or running (`{0}`), so Memory Integrity and Credential Guard were not the cause.
- The `VirtualMachinePlatform` feature was still enabled, and WSL was on version 2.
- The registry value `EnableVirtualizationBasedSecurity` was `1`.
- The PC was not domain-joined, not Azure AD-joined, and had no MDM URL, so no company policy was enforcing VBS.
- Docker was not installed.

**What I tried:**
1. `bcdedit /set hypervisorlaunchtype off`
2. Disabled the Virtual Machine Platform feature.
3. Set the DeviceGuard registry values to `0`, and the Group Policy value to `0`.
4. Restarted after each change and re-checked.

**What I found:** disabling Virtual Machine Platform removed my `hypervisorlaunchtype off` entry from the boot configuration, so I had to re-apply it. After re-applying, the following were all verified: `hypervisorlaunchtype = Off`, policy `EnableVirtualizationBasedSecurity = 0`, HVCI `Enabled = 0` and `Locked = 0`. VBS status was still `2`. Smart App Control reads as enabled (`VerifiedAndReputablePolicyState = 1`). I did not prove a link between Smart App Control and VBS, and I did not turn it off, because that change is permanent until Windows is reinstalled.

**Decision:** I timeboxed the problem at three attempts and continued on the Windows Hypervisor Platform backend. I will judge performance when the first VM is built.

### 2. Lab subnet overlapped my home network

**Symptom:** `ipconfig` showed my physical Ethernet adapter at `192.168.100.31` with gateway `192.168.100.1`, and the VMnet1 host adapter also at `192.168.100.1`.

**Cause:** I chose `192.168.100.0/24` for the lab without checking my home network. Two interfaces in the same subnet, one holding the router's address, can cause routing to the wrong interface.

**Fix:** moved VMnet1 to `10.10.100.0/24`. No VMs existed yet, so nothing had to be rebuilt. 
### 3. Git repository created in the wrong folder

**Cause:** the target folder did not exist, so `cd` failed and PowerShell stayed in `C:\Windows\System32`. `git init` and `mkdir` then ran there.

**Fix:** deleted the `.git` folder and the empty `docs` folders, and confirmed with `Test-Path`. 
### 4. First `git push` failed

**Cause:** the repository had no commits (the files had not been created yet), and the remote URL contained the literal placeholder `<your-username>`.

**Fix:** created the files, committed, and corrected the URL with `git remote set-url`.

### Open issue: `D:\Lab` disappeared

`Get-ChildItem D:\Lab` listed my folders at first, then later reported that the path did not exist, while `C:\VMs` was unaffected. Cause not yet determined. 

## Lessons learned

- Windows security features are layered, each with its own switch. Verify the state after every change, because a later change can undo an earlier one (removing Virtual Machine Platform reset my boot setting).
- Timebox troubleshooting. After three attempts with no new evidence, document the state and continue.
- Check the addressing of your real network before choosing lab subnets.
- Before any command that writes, verify the working directory (`pwd`).
- Read error messages literally. Both Git errors described the exact problem.