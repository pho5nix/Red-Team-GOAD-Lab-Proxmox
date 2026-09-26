#  Verify

1. Proxmox: all 5 GOAD VMs, plus ELK if built, running with the reference table IPs.
2. From the provisioning host:

```bash
ping 192.168.56.10
nslookup sevenkingdoms.local 192.168.56.10
nslookup essos.local 192.168.56.10
```

3. From your desktop, direct RDP to kingslanding at 192.168.56.10 with the domain admin credentials GOAD sets. Pull current defaults from your cloned repo's GOAD vars or README.
4. Confirm the cross-forest trust, from a sevenkingdoms host resolve and authenticate toward essos.local resources.
5. From Kali confirm both paths work. The direct VLAN 150 to VLAN 56 path and the redirected VLAN 150 to VLAN 160 to VLAN 56 path once your redirectors are configured.

---

# Lock down and snapshot

## Remove build time exceptions

Three temporary allowances were made for the Packer and provisioning phase.

- Build time internet, Part 1 AD_LAB rule 1: disable or delete it. VLAN 56 should originate nothing outbound.
- DHCP on AD_LAB, Part 6: disable it again in Services -> DHCP Server -> AD_LAB.

## Snapshot everything

Once verified, snapshot all 5 GOAD VMs.

```bash
qm snapshot <vmid> clean-baseline
```

Also snapshot Kali and the Sliver server once tooled up. If you built ELK, snapshot it too, though its indices grow over time, so you may prefer to keep ELK running rather than reverting it and only revert the GOAD Windows VMs between exercises.

---

# All errors, issues and troubleshooting until the working build

- Windows eval ISO URLs rot. If Packer's ISO fetch fails, recheck the current Microsoft eval URL.
- Packer and Terraform var file formats drift between GOAD releases. Check against your freshly cloned template files.
- Nested virtualization or MTU mismatches can cause intermittent WinRM or Ansible timeouts on bare metal Proxmox plus pfSense. Check MTU consistency across the VLAN aware bridge and the pfSense VLAN interfaces before assuming a GOAD bug.
- DNS confusion: if AD behaves oddly, confirm the Windows hosts point DNS at the DCs, not pfSense.
- ELK OOM(out of memory) or will not start: Elasticsearch needs its roughly 8 GB and vm.max_map_count=262144.
- ELK inventory group mismatch: elk.yml references a specific elk inventory group and variables. Check your inventory entry against what the current elk.yml expects.
- Ubuntu installer plus isolated VLAN, "Temporary failure resolving" or IPv6 "Network is unreachable" on the mirror screen: work through Part 4's list in order, DNS, firewall rule scope, NAT, IPv6.
- ansible-core versus modern Python: GOAD's docs pin `ansible-core==2.12.6`, which crashes on Python 3.12 and newer. Install `ansible-core>=2.20` instead, see Part 4.
- setup_proxmox.sh run from the wrong directory produces a "requirement file does not exist" error. Always cd to the GOAD repo root first.
- SPICE clipboard never works on a headless server VM, since it requires an active X11 session. Use SSH instead.
- Provisioning VM cannot reach Proxmox's management IP, SCP or API calls hang or time out. Add the explicit allow rule from Part 5.0, above the RFC1918 block rule.
- proxmox_pool equals Templates has no default and the pool does not exist yet. Create the pool and add Pool.Allocate to the role.
- iso_file in the pkvars.hcl expects a specific filename that will not match Microsoft's actual filename. Rename the uploaded ISO to match, or edit the field.
- 403 Permission check failed on an sdn zones localnetwork path when Packer creates the VM. Add SDN.Use permission to the role, even without touching Proxmox's actual SDN feature.
- Unsupported format qcow2 errors from LvmThinPlugin: LVM-thin storage cannot store qcow2 disks. Set vm_disk_format to raw.
- Volume local:iso/virtio-win.iso does not exist: upload the VirtIO driver ISO as described in Part 6.
- Error starting VM, bridge does not exist: the network_adapters block in packer.json.pkr.hcl ships with hardcoded values. Edit it to your real bridge and VLAN, see Part 6.
- Windows VM gets an APIPA address and WinRM never becomes reachable: temporarily enable DHCP on AD_LAB during the build phase, see Part 6.
- Error getting WinRM host, 500 QEMU guest agent is not running, even though qm agent ping succeeds as root on the Proxmox host: this is almost always the missing VM.GuestAgent.Audit and VM.GuestAgent.Unrestricted permissions. Check Part 5 before reinstalling the guest agent manually.
- MSSQL installer download fails with a 404 on download.microsoft.com: the direct CDN link hardcoded in ansible/roles/mssql/defaults/main.yml has rotted, the same category of issue as the Windows Server eval ISOs. Replace it with Microsoft's stable fwlink, https://go.microsoft.com/fwlink/?linkid=866658, and rerun. See Part 7.
- mssql role's Install the database task fails once or twice then succeeds on retry: commonly a pending reboot or timing race on a freshly domain joined server settling out between attempts. Let Ansible's built in retries run before intervening.
- SSMS install on member servers hangs indefinitely rather than just running slowly: confirmed as a known open issue in the official GOAD repo. Ansible's WinRM become mechanism has no timeout and the installer can silently block on a dialog. If a Task Manager check on the VM shows genuinely zero activity for 15+ minutes, stop waiting and skip the play with --skip-tags mssql_ssms via a direct ansible-playbook call, see Part 7. SSMS is a GUI tool only, not part of the lab's vulnerable attack surface, safe to skip.
- goad.sh -t status fails with 403 Forbidden, Permission check failed on /pool/Templates, Pool.Audit: add Pool.Audit to the GOADProvisioner role, see Part 5. Not needed by Packer or Terraform, only by GOAD's own status command.
- ansible-playbook: command not found when run with sudo, but works without it: sudo does not carry over the Python virtual environment's PATH. Run commands as the actual root shell without sudo or activate the venv first with source /root/GOAD/.venv/bin/activate.
- provision_extension or install_extension reports extension not enabled in instance: the instance's own extensions list, in workspace/<instance-id>/instance.json, starts empty regardless of what list_extensions shows. Add the extension name to that array directly before it can be provisioned.
- Elasticsearch apt key task fails, apt-key executable not found: Ubuntu removed the apt-key binary entirely starting with 22.04, and no package restores it. Patch the role to use ansible.builtin.get_url plus a signed-by apt_repository line instead, the same category of fix as the MSSQL URL rot elsewhere in this guide.
- curl https://sliver.sh/install | sudo bash only produces a sliver client, no sliver-server binary anywhere: confirmed on an actual install. Download both binaries directly from the latest GitHub release instead, see Part 3.

---

