# GOAD provisioning host (VLAN 56)

The provisioning host orchestrates Packer, Terraform and Ansible. It sits on VLAN 56 so it can talk to the Windows VMs it builds. Host needs outbound internet during the build. Keep the build time WAN allow rule and the Outbound NAT rule from Part 1 enabled all the way through the Terraform and Ansible install, removing both only once the full lab is verified working.

## Create the VM

- Create VM, attach Ubuntu Server ISO, NIC tag:160, vCPU:1 socket-4 cores, RAM:4 GB, Storage:40 GB.
- During install, assign a static IP at Ubuntu network menu inside 192.168.56.0/24 that does not collide with the GOAD hosts, for example 192.168.56.200 with gateway 192.168.56.1. DHCP is off on this VLAN, so set this manually in the installer.

### Ubuntu installer network notes on this VLAN

Work through the below if you hit "Temporary failure resolving" or "Network is unreachable" on the mirror screen.

1. Nameserver: use pfSense's gateway 192.168.56.1, not a public resolver directly. Confirm pfSense's own DNS Resolver is enabled and listening on the AD_LAB interface.
2. The build time firewall rule must allow any port, not just 443. The installer needs 53 for DNS and 80 for the HTTP mirror too.
3. Outbound NAT must exist for VLAN 56 to WAN. An allow rule alone does not let return traffic route back. Without NAT you will see connections time out.
4. IPv6: if the installer tries bracketed IPv6 addresses and fails with "Network is unreachable," your VLAN has no IPv6 routing, which is expected. Either set the mirror address to http://ipv4.archive.ubuntu.com/ubuntu/ to force IPv4 only DNS answers, or skip the mirror and fix IPv6 after boot.
5. If all else fails, hit Done on the mirror screen and finish the install. You can apt update normally once booted, after applying the fixes above.

## Log in as root

This VM is a throwaway orchestration host inside an isolated lab VLAN, not a production server.  Every tool in this build assumes root.

```bash
sudo -i
```

Run everything from here on as root, in the same shell.

## Disable IPv6 permanently if you hit the installer IPv6 issue above

```bash
tee /etc/sysctl.d/99-disable-ipv6.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sysctl --system
ip addr show | grep inet6
```

The last command should return nothing.

## Remote access to this VM, use SSH

This is a headless server with no X11 session. SPICE's clipboard feature requires an active X11 session to function, so it will never work here regardless of configuration.

```bash
apt install -y openssh-server
systemctl enable --now ssh
```

From your desktop, `ssh root@192.168.56.5`. This gives native copy paste and scroll through your own terminal. Install tmux and run all long running steps inside it, so a dropped SSH session does not kill an in progress build.

```bash
apt install -y tmux
tmux new -s goad
# reattach anytime with: tmux attach -t goad
```

## Fix the locale

If `./goad.sh` errors with "Ansible could not initialize the preferred locale":

```bash
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 LANGUAGE=en_US.UTF-8
export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 LANGUAGE=en_US.UTF-8
echo 'export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 LANGUAGE=en_US.UTF-8' >> /root/.bashrc
```

Verify with `locale`. All lines should read en_US.UTF-8.

## Install dependencies

```bash
apt update && apt install -y git python3 python3-venv python3-pip sshpass rsync openssh-client
```

## Clone GOAD repository

```bash
cd /root
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
```

## Edit setup_proxmox.sh before running it for Python compatibility

GOAD's shipped script pins `ansible-core==2.12.6`, which predates Python 3.12 and crashes with error `'_AnsiblePathHookFinder' object has no attribute 'find_spec'` on modern Python. If your Ubuntu install shipped Python 3.12 or newer, this pin will always break.

Open scripts/setup_proxmox.sh and modify the line:

```bash
python3 -m pip install ansible-core==2.12.6
```

to:

```bash
python3 -m pip install "ansible-core>=2.20"
```

2.20 is the first release with official Python 3.14 controller support.

## Run the prerequisite script, from the repo root

```bash
cd /root/GOAD
bash ./scripts/setup_proxmox.sh
```

**Run this from the repo root and not from inside scripts directory**, since it uses relative paths. This installs Packer, Terraform and the Ansible venv. Then runs ansible-galaxy install against ansible/requirements.yml.

Verify:

```bash
source .venv/bin/activate
ansible --version
packer version
terraform version
```

## Launch GOAD's interactive menu

```bash
./goad.sh
```

Should launch cleanly with no locale or Ansible import errors. Exit the menu once confirmed. You will come back to run installs in Part 6.

---
