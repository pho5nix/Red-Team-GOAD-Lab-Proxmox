## Get the ISO files into Proxmox

You need three ISO files uploaded to Proxmox before building.

**1. Windows Server 2019 and 2016 evaluation ISOs.** Use https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019 and the equivalent 2016 page, register, and download.

**Before uploading**, check what filename GOAD's `pkvars.hcl` actually expects. It may reference a specific filename that will not match the long build numbered filename Microsoft gives you. Either rename the file to match before uploading, or edit the iso_file line to match what you upload.

**2. VirtIO driver ISO, required.** GOAD's Packer config references local:iso/virtio-win.iso for the Windows virtio disk and network drivers needed by KVM. Download the current stable build:

```
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

Upload it to Proxmox as exactly "virtio-win.iso".

Upload all ISOs via the Proxmox UI. Node -> local storage -> ISO Images, Upload. Uploading from your desktop, which has full internet, is faster than routing large files through VLAN 56's temporary WAN window.

Verify all three landed correctly:

```bash
ssh root@<proxmox-management-ip> "ls -la /var/lib/vz/template/iso/"
```

## Build the autounattend and cloud-init ISOs

Run this script that builds everything.

```bash
cd /root/GOAD/packer/proxmox/
./build_proxmox_iso.sh
```

This single run builds a per-OS autounattend ISO for every Windows variant GOAD supports and automatically writes the correct SHA-256 checksum into each `pkvars.hcl` file. It also builds one general `scripts_withcloudinit.iso`, which injects Cloudbase-Init and needs manual transfer to Proxmox.

Windows 11 lines may error in some clones with missing answer files. This does not affect the 2019 and 2016 lines, which complete before that section runs.

Verify what actually got built:

```bash
ls -la ./iso/
```

You should see Autounattend_winserver2019_cloudinit.iso, Autounattend_winserver2016_cloudinit.iso and scripts_withcloudinit.iso.

Only `scripts_withcloudinit.iso` needs manual transfer. The per-OS autounattend ISOs are referenced by Packer via a local relative path and Packer uploads them to Proxmox itself during the build.

```bash
scp ./iso/scripts_withcloudinit.iso root@<proxmox-management-ip>:/var/lib/vz/template/iso/
```

This runs on the provisioning VM, pushing to Proxmox's management IP. Expect a host key confirmation prompt and a password prompt the first time.

Verify the transfer landed completely:

```bash
ssh root@<proxmox-management-ip> "ls -la /var/lib/vz/template/iso/scripts_withcloudinit.iso"
```

Compare the byte size against your local `ls` output. It should match exactly.

## Configure Packer variables

GOAD's Proxmox Packer setup splits configuration across two files per OS build. Inspect your actual clone rather than trust field names from elsewhere.

```bash
cd /root/GOAD/packer/proxmox/
ls *.pkvars.hcl *.template
cat variables.pkr.hcl
```

**File 1 - windows_server2019_proxmox_cloudinit.pkvars.hcl (and the 2016 equivalent), OS and VM specific settings:**

```
winrm_username        = "vagrant"
winrm_password        = "vagrant"
vm_name               = "WinServer2019x64-cloudinit"
template_description  = "..."
iso_file              = "local:iso/<your-uploaded-2019-iso-filename>"
autounattend_iso      = "./iso/Autounattend_winserver2019_cloudinit.iso"
autounattend_checksum = "sha256:<auto populated by build_proxmox_iso.sh>"
vm_cpu_cores          = "2"
vm_memory             = "4096"
vm_disk_size          = "40G"
vm_sockets            = "1"
os                    = "win10"
vm_disk_format        = "raw"
```

winrm_username and winrm_password are the local Windows admin account the unattended install creates, not your Proxmox credentials. Leave them as vagrant unless you have a reason to change them.

**Set vm_disk_format to raw**, not qcow2 if your VM storage pool is local-lvm. LVM-thin storage cannot store qcow2 images at all. qcow2 only works on file based storage such as directory, NFS or CIFS. Do this in both the 2019 and 2016 `pkvars.hcl` files.

Do not hand-edit autounattend_checksum. build_proxmox_iso.sh writes it automatically each time it regenerates the ISO.

Only iso_file and vm_disk_format typically need editing here (you can also edit the vm_name - not affect anything in build). Everything else ships correct.

**File 2 - config.auto.pkrvars.hcl. Copy from the template, the Proxmox connection itself:**

```bash
cp config.auto.pkrvars.hcl.template config.auto.pkrvars.hcl
```

Fill in:

```
proxmox_url             = "https://<your-proxmox-management-ip>:8006/api2/json"
proxmox_username        = "goad-provisioner@pve"
proxmox_password        = "<password set via pveum passwd in Part 5>"
proxmox_skip_tls_verify = "true"
proxmox_node            = "<exact node name from Proxmox UI, top left>"
proxmox_pool            = "Templates"
proxmox_iso_storage     = "local"
proxmox_vm_storage      = "local-lvm"
```

Use the plain username and password we created here. variables.pkr.hcl declares only proxmox_username and proxmox_password, no token style variable exists in this GOAD version and Terraform ends up using the same username and password auth later, so this one credential covers both tools.

## Fix the hardcoded network adapter in packer.json.pkr.hcl

The network_adapters block inside the source block is not parameterized. It ships with hardcoded values that will not match your environment, for example a bridge and VLAN tag that do not exist in this build's network.

```bash
cd /root/GOAD/packer/proxmox/
grep -A4 "network_adapters" packer.json.pkr.hcl
```

Edit the block to match your actual setup:

```
network_adapters {
  bridge   = "vmbr0"
  model    = "virtio"
  vlan_tag = "56"
}
```

Use VLAN 56, since the temporary build VM needs to be reachable by the provisioning VM's WinRM connection during install and VLAN 56 is the network the rest of this build is already set up around. This is a direct file edit not a `pkvars.hcl` variable.

## Temporarily enable DHCP on VLAN 56, required for template builds

This conflicts with Part 1's design on purpose. The finished GOAD lab uses static IPs from Terraform but the temporary build VM Packer creates has no such static config yet. It is a stock Windows install expecting DHCP like any other. Without it, Windows self assigns an unroutable APIPA address and Packer can never connect.

pfSense: Services -> DHCP Server -> AD_LAB tab.  Enable DHCP with a small temporary range that avoids the final static GOAD hosts and your provisioning VM:

```
Range: 192.168.56.100 to 192.168.56.120
```

Leave this enabled through both template builds(2019 and 2016), then disable it again once both templates exist and **before running the actual GOAD install (Part 6)**.

## Edit the build block in packer.json.pkr.hcl

```hcl
build {
  sources = ["source.proxmox-iso.windows"]

  provisioner "powershell" {
    elevated_password = "vagrant"
    elevated_user     = "vagrant"
    inline            = ["Set-MpPreference -DisableRealtimeMonitoring $true"]
  }

  provisioner "powershell" {
    elevated_password = "vagrant"
    elevated_user     = "vagrant"
    scripts           = ["${path.root}/scripts/sysprep/cloudbase-init.ps1"]
  }

  provisioner "powershell" {
    elevated_password = "vagrant"
    elevated_user     = "vagrant"
    pause_before      = "1m0s"
    scripts           = ["${path.root}/scripts/sysprep/cloudbase-init-p2.ps1"]
  }
}
```

Replace the build block with the above.

Notes on this block:

- The first provisioner disables Windows Defender real time scanning before anything else runs. Real time AV scanning of freshly written PowerShell files can interfere with the provisioner's file upload and execution sequence, so this removes one contributing factor to occasional upload timing issues.
- Elevation, elevated_user and elevated_password, stays on for all three steps. Windows applies remote UAC token filtering to local, non domain accounts over WinRM, so a non elevated session can run with a standard user token even if vagrant is a local admin. The MSI install and sysprep both need real admin rights.

## Build each template

```bash
cd /root/GOAD/packer/proxmox/
packer init .
packer build -var-file=windows_server2019_proxmox_cloudinit.pkvars.hcl .
packer build -var-file=windows_server2016_proxmox_cloudinit.pkvars.hcl .
```

Watch the first 30 seconds of output closely. Authentication problems, wrong password, wrong node name, missing pool or guest agent permission, surface immediately. Networking, DHCP and guest agent problems surface during the "Waiting for WinRM" stage.

Record the resulting template VMIDs, shown as "A template was created: id" at the end of a successful run. You need these for Terraform in Part 6.

---
