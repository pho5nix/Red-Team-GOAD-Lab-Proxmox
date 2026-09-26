## Firewall exception - Provisioning VM must reach Proxmox's management IP address

The provisioning VM, 192.168.56.200 needs to reach your Proxmox host's management IP for SCP transfers and for every Packer and Terraform API call. 
Part 1's AD_LAB rules block all RFC1918 traffic by default except the gateway and Proxmox's management IP is a RFC1918 network.

Add this rule on the AD_LAB tab, above the RFC1918 block rule:

- Allow AD_LAB subnet to your Proxmox management interface IP address, ports 22 and 8006.
- Description: Provisioning VM to Proxmox management.
- Save and Apply changes.

## Create the role in Proxmox (9.x)

Datacenter -> Permissions -> Roles -> Create, Role ID: GOADProvisioner.

Check these privileges:

```
Datastore.AllocateSpace, Datastore.AllocateTemplate, Datastore.Audit,
VM.Allocate, VM.Audit, VM.Clone, VM.Migrate, VM.PowerMgmt,
VM.Config.CDROM, VM.Config.Cloudinit, VM.Config.CPU, VM.Config.Disk,
VM.Config.HWType, VM.Config.Memory, VM.Config.Network, VM.Config.Options,
Pool.Allocate, Pool.Audit, SDN.Use, VM.GuestAgent.Audit, VM.GuestAgent.Unrestricted
```

Pool.Audit was found necessary after the build, needed for goad.sh's own status command to read pool membership. 

Notes for permissions:

- Pool.Allocate is required because GOAD's Packer config references a mandatory Proxmox resource pool named Templates.
- SDN.Use is required. Even a plain, non SDN VLAN aware bridge with a VLAN tag gets checked by Proxmox 8.x and 9.x against an implicit localnetwork SDN zone at creation time. This shows up as `403 Permission check failed (/sdn/zones/localnetwork/<bridge>/<vlan>, SDN.Use)`. It is unrelated to Proxmox's actual optional SDN subsystem.
- VM.GuestAgent.Audit and VM.GuestAgent.Unrestricted are required. Since VLAN 56's DHCP is handled externally by pfSense rather than Proxmox itself, Packer has no way to learn a template build VM's IP address except by asking the Proxmox API to query the QEMU Guest Agent inside the guest. Without these two privileges that API call is rejected with a 403, which Packer's log surfaces as the misleading message "500 QEMU guest agent is not running" even when the agent is actually running fine. Check this permission before assuming the guest agent itself needs reinstalling.

With this role granted at path / with Propagate checked, all of the above cascades down automatically. No separate ACL entries needed.

## Create the user in the pve realm

Datacenter -> Permissions -> Users, Add.

- User name: goad-provisioner
- Realm: pve, not pam. pam maps to a real Linux system account on the Proxmox host, which this service account does not need. pve is Proxmox's own internal user database, the correct choice for an automation account.

Result: goad-provisioner@pve.

## Set a password on the user - required for Packer

GOAD's Packer configuration for Proxmox only supports username and password authentication. Its variables `.pkr.hcl` declares `proxmox_username` and `proxmox_password` with no token style variable.

The password field is not in the Edit dialog, it is a separate toolbar action.

GUI: Datacenter -> Permissions -> Users, click the goad-provisioner@pve row to select it, click the Password button in the toolbar, set it.

CLI, more reliable.  On the Proxmox host shell:

```bash
pveum passwd goad-provisioner@pve
```

Prompts interactively for the new password.

**Keep this password.** Both Packer and Terraform authenticate with plain username and password in this GOAD version, confirmed once the actual build reached that point, so this is the only credential you need to create here, no API token required.

## Create the Templates pool

This is required because Packer's config hardcodes `proxmox_pool` equals `Templates` with no default value.

Datacenter -> Permissions -> Pools -> Create, Pool ID(Name): Templates.

## Grant the role, at root path /

Datacenter -> Permissions -> Add.

| Path | User | Role |
|---|---|---|
| / | goad-provisioner@pve | GOADProvisioner |

Use path / at root, not a scoped path like /vms. The role mixes VM privileges and datastore privileges which live in different parts of the permission tree and Terraform will be creating brand new VMIDs that cannot be pre-scoped. Propagate ensures this also covers the Templates pool.

---
