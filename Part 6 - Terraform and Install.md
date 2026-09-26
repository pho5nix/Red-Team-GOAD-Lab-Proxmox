## Configure GOAD's global config file

Configuration for the Proxmox provider lives in a global file `goad.ini` and GOAD's Python layer generates the Terraform files from a shared template at run time. Check this against your own clone before assuming the file exist:

```bash
find / -iname "goad.ini" 2>/dev/null
```

This resolves to a fixed path under your home directory, commonly /root/.goad/goad.ini. If it does not exist yet, run `./goad.sh` once first, since GOAD creates a default copy of this file the first time it runs.

Open it and add or edit these sections. The proxmox and proxmox_templates_id sections do not come filled in with real values by default, only placeholders, so all of these need setting explicitly:

```ini
[default]
lab = GOAD
provider = proxmox
provisioner = local
ip_range = 192.168.56

[proxmox]
pm_api_url = https://<your-proxmox-management-ip>:8006/api2/json
pm_user = goad-provisioner@pve
pm_pass = <the password set via pveum passwd in Part 5.3>
pm_node = <your real Proxmox node name>
pm_pool = Templates
pm_full_clone = false
pm_storage = local-lvm
pm_vlan = 56
pm_network_bridge = vmbr0
pm_network_model = virtio

[proxmox_templates_id]
WinServer2019_x64 = <your 2019 template VMID from Part 5>
WinServer2016_x64 = <your 2016 template VMID from Part 5>
```

Notes on values that differ from GOAD's own defaults, since getting these wrong is easy to miss:

- pm_pool must match the pool we actually created and granted Pool.Allocate on in Part 5, not the default's GOAD value.
- pm_storage must match your actual VM storage pool, local-lvm in this build, not the default's local.
- pm_network_model should be virtio not the default's e1000, to match every other VM in this build.
- pm_vlan should match your actual AD_LAB VLAN tag. The guide uses VLAN 56 throughout.
- pm_pass has no default in GOAD's own config generator, since it is a secret, so it must always be added manually.
- Confirm your real template VMIDs directly rather than assume, since Packer's auto-numbering can reuse IDs across rebuilds:

```bash
ssh root@<proxmox-management-ip> "qm list"
```

## Check

```bash
cd /root/GOAD
./goad.sh -t check -l GOAD -p proxmox
```

Resolve every failing line before installing.

## Before installing, patch the MSSQL installer URL

GOAD's mssql role has a hardcoded Microsoft download link that has gone dead. Confirmed by an actual build hitting a 404:

```bash
grep download_url_2019 /root/GOAD/ansible/roles/mssql/defaults/main.yml
```

If it shows the old direct CDN link replace it with Microsoft's stable fwlink redirect, which is designed not to rot the way raw CDN paths do:

```bash
sed -i 's|^download_url_2019:.*|download_url_2019: https://go.microsoft.com/fwlink/?linkid=866658|' /root/GOAD/ansible/roles/mssql/defaults/main.yml
grep download_url_2019 /root/GOAD/ansible/roles/mssql/defaults/main.yml
```

Verify it actually resolves before trusting it:

```bash
curl -IL "https://go.microsoft.com/fwlink/?linkid=866658"
```

Confirm a redirect chain ending in 200 OK with a real .exe as the final target.

## Install

```bash
./goad.sh -t install -l GOAD -p proxmox
```

This runs Terraform init, plan and apply using the config generated from goad.ini, cloning templates into 5 VMs with static IPs, then the full Ansible provisioning. This promotes the DCs, builds sevenkingdoms.local with child domain north.sevenkingdoms.local and a separate essos.local forest. Sets the cross-forest trust, installs MSSQL, IIS and ADCS on the member servers and seeds the intentional vulnerabilities. Expect 45 or more and be patient.

Terraform will prompt interactively for pm_password even though it is already set in goad.ini, since the variable is declared as sensitive with no default and is not auto-injected as an environment variable in this version. Enter the same password from goad.ini when prompted.

Confirmed during an actual run: the AD_LAB VMs themselves need working outbound internet during this phase, not just the provisioning host. The MSSQL installer you download  is only a small bootstrap stub, its real job is downloading the full multi hundred MB installation media from Microsoft from inside the target VM itself. Keep the build time WAN allow and Outbound NAT rules from Part 1 enabled through the entire install, do not close them early.

**One slow, quiet step is normal, not stuck:**

- The mssql role's Install the database task can fail once or twice and retry automatically before succeeding, commonly caused by a pending reboot on a freshly domain joined server settling itself out between attempts. Let Ansible's built in retries run before assuming a real failure.

**One step is a known, currently open bug in GOAD itself, not something to wait out:**

- SSMS install on the member servers can hang indefinitely, not just run slowly. This is confirmed as a known issue, filed against the official GOAD repo and traced to two combined causes. Ansible's WinRM become mechanism has no built in timeout, so if the underlying installer blocks on anything the task waits forever with no error. Separately, unattended Windows installers can silently stall on a dialog box that never gets clicked, commonly a certificate check or a build Microsoft has marked unsupported.

Check the VM's own Task Manager for the SSMS_installer running with zero CPU activity or if this task sits still for more than about 15 minutes. If it is stuck rather than just slow do not wait longer, skip the play instead. SSMS is a GUI convenience tool, not part of the lab's actual vulnerable attack surface, the database engine, the linked servers and their credential mappings are configured by an earlier, separate part of the same role and are unaffected by skipping SSMS.

To skip it, first stop the hung task, either Ctrl+C the terminal running install, or reboot the affected VM directly from Proxmox if Windows itself will not respond to a normal shutdown:

```bash
ssh root@<proxmox-management-ip> "qm reboot <vmid>"
```

Then run the same playbook the wrapper itself uses, with the one tag skipped. This calls ansible-playbook directly with the exact inventory files GOAD's install log already showed it using:

```bash
cd /root/GOAD/ansible
source /root/GOAD/.venv/bin/activate
ansible-playbook \
  -i /root/GOAD/ad/GOAD/data/inventory \
  -i /root/GOAD/workspace/<your-instance-id>/inventory \
  -i /root/GOAD/globalsettings.ini \
  servers.yml \
  --skip-tags mssql_ssms
```

Run this as the actual root shell, not with sudo in front of it, since sudo does not carry over the Python virtual environment's PATH and will report ansible-playbook as not found even though it works fine without sudo.

If something fails partway, usually from VM reboot timing, you can rerun just the Ansible portion without recreating VMs. This GOAD version requires both an instance ID and a value for the ansible only flag:

```bash
./goad.sh -t install -l GOAD -p proxmox -i <your-instance-id> -a yes
```

Find your instance ID from the table GOAD printed when it first created the lab or check inside the interactive menu if you did not record it.

## Check the result

```bash
./goad.sh -t status -l GOAD -p proxmox -i <your-instance-id>
```

This prints a table of every VM with its hostname, running state and current IP address. A fast, reliable way to confirm all 5 GOAD hosts are up with the correct addresses. The two Packer templates will correctly show stopped with no IP, that is expected as templates are never meant to run.

Once the full install completes and you have verified the lab, close the build time internet exception rule for AD_LAB and the temporary DHCP range.  
Keep the exceptions still active if you want to setup the ELK covered next. Close them after ELK SIEM setup

---


# Optional ELK SIEM

Current GOAD versions wire ELK in as a formal extension. This turns the range into a purple team setup:  You run an attack from Kali, then check Kibana to see what telemetry it generated.

## Build the ELK VM

The extension's own folder, extensions/elk/providers/, has automation for ludus, aws, vmware and azure, but no proxmox folder. On this provider there is no automated VM creation for this extension, building it by hand is the expected path.

- Create VM, attach Ubuntu Server ISO, NIC tag:56, vCPU:1 socket-4cores, RAM: 8 GB, Storage: 80 GB.
- Static IP: 192.168.56.50, gateway: 192.168.56.1. Use the gateway as DNS too, matching every other VM in this lab. Confirmed there is no dependency on AD domain name resolution anywhere in the extension's role or playbook and GOAD's own inventory defaults every VM to the gateway for DNS regardless.
- Elasticsearch needs a raised map count:

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elastic.conf
sudo sysctl --system
```

## Set up a dedicated SSH account for Ansible

Unlike every Windows host so far, this extension connects over plain SSH and root login should stay disabled. Create a dedicated automation account rather than opening up your own login. Key only, no password, sudo scoped to just this account.

On the ELK VM:

```bash
sudo useradd -m -s /bin/bash ansible
```

On the provisioning VM, generate a key pair used only for this:

```bash
ssh-keygen -t ed25519 -f /root/.ssh/goad_elk_ansible -N ""
ssh-copy-id -i /root/.ssh/goad_elk_ansible.pub ansible@192.168.56.50
```

Back on the ELK VM, scope passwordless sudo to this one account and lock its password entirely:

```bash
echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible-elk
sudo chmod 440 /etc/sudoers.d/ansible-elk
sudo passwd -l ansible
```

Verify both key login and passwordless sudo work from the provisioning VM:

```bash
ssh -i /root/.ssh/goad_elk_ansible ansible@192.168.56.50 "sudo whoami"
```

Should print root with no prompts at all. Fix anything above before continuing if it asks for a password anywhere in this chain.

## Enable the extension on this instance

list_extensions in GOAD's interactive console shows every extension available in the repo, but an instance also tracks its own enabled list separately and a freshly created instance starts with none. Trying to provision an extension before this returns extension not enabled in instance.

```bash
cd /root/GOAD
sed -i 's/"extensions": \[\]/"extensions": ["elk"]/' workspace/<your-instance-id>/instance.json
cat workspace/<your-instance-id>/instance.json
```

Confirm it now shows elk in the extensions array.

## Point the extension's inventory template at your real account and IP

The extension's own inventory file, `extensions/elk/inventory`, ships with an unqualified connection line, an unfilled `ip_range placeholder` and has no ansible_user, since it assumes root login by default. Edit the template directly:

```bash
grep "^elk " /root/GOAD/extensions/elk/inventory
sed -i "s/^elk ansible_host=/elk ansible_user=ansible ansible_ssh_private_key_file=\/root\/.ssh\/goad_elk_ansible ansible_host=/" /root/GOAD/extensions/elk/inventory
grep "^elk " /root/GOAD/extensions/elk/inventory
```

Confirm the second grep shows ansible_user and the key path added alongside the existing ansible_host and connection settings.

The per-instance rendered copy of this file that provision_extension actually reads does not get generated automatically on this provider, since that render step assumes a provider that provisions the ELK VM itself. Create it manually, substituting the real subnet for the placeholder:

```bash
sed 's/{{ip_range}}/192.168.56/g' /root/GOAD/extensions/elk/inventory > /root/GOAD/workspace/<your-instance-id>/elk_inventory
cat /root/GOAD/workspace/<your-instance-id>/elk_inventory
```

Confirm the output shows 192.168.56.50 and your account and key with no placeholder text left anywhere.

## Patch the role for modern Ubuntu, apt-key is gone

Ubuntu removed the apt-key binary entirely starting with 22.04. If your ELK VM is on a release that new or newer, the elk role's key import task will fail immediately with a message about apt-key not being found. Check first:

```bash
ssh -i /root/.ssh/goad_elk_ansible ansible@192.168.56.50 "lsb_release -a"
grep -n "apt_key:" /root/GOAD/extensions/elk/ansible/roles/elk/tasks/main.yml
```

If lsb_release shows 22.04 or newer and the grep finds a match, patch it. Open the file and replace this block:

```yaml
- name: Add Elasticsearch apt key.
  apt_key:
    url: https://artifacts.elastic.co/GPG-KEY-elasticsearch
    state: present

- name: Add Elasticsearch repository.
  apt_repository:
    repo: 'deb https://artifacts.elastic.co/packages/{{ elasticsearch_version }}/apt stable main'
    state: present
    update_cache: true
```

with the modern keyring based equivalent, which downloads the key to a file and references it explicitly instead of relying on the removed apt-key command:

```yaml
- name: Add Elasticsearch apt key.
  ansible.builtin.get_url:
    url: https://artifacts.elastic.co/GPG-KEY-elasticsearch
    dest: /usr/share/keyrings/elasticsearch-keyring.asc
    mode: '0644'

- name: Add Elasticsearch repository.
  apt_repository:
    repo: 'deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.asc] https://artifacts.elastic.co/packages/{{ elasticsearch_version }}/apt stable main'
    state: present
    update_cache: true
```

This fix covers Elasticsearch, Kibana and Logstash together, confirmed only one apt_key task exists in the whole role since all three packages come from the same Elastic repository.

## Run it

From inside GOAD's interactive console:

```bash
cd /root/GOAD
./goad.sh -i <your-instance-id>
```

```
provision_extension elk
```

**Use provision_extension, not install_extension**, since install_extension also attempts a providing step this provider has no automation for and would likely fail before reaching Ansible. Your VM already exists, only the Ansible install is needed.

This installs Elasticsearch, Kibana and Logstash on the ELK VM, then installs the log shipping agent on all 5 domain joined Windows hosts automatically, the extension's own inventory maps that log agent role to the same domain group already used everywhere else in the lab. Expect several minutes as Elasticsearch's first startup is slow.

## Verify

From your desktop, VLAN 100, browse to:

```
http://192.168.56.50:5601
```

Should load Kibana. If it does not, check the services directly:

```bash
ssh -i /root/.ssh/goad_elk_ansible ansible@192.168.56.50 "sudo systemctl status elasticsearch kibana --no-pager"
```

Confirm data is arriving by checking for a recent winlogbeat index in Kibana's index management or discover view. Give it a few minutes after install finishes if nothing shows up immediately.

No firewall changes are needed for any of this. ELK lives on VLAN 56 like everything else in the lab, your desktop reaching port 5601 is already covered by the existing desktop to AD_LAB allow and ELK needs no internet access after installation completes.


**Since the Lab setup is finished and all tests completed with success, now is time to close the build time internet exception rule for AD_LAB and the temporary DHCP range**


## Purple team drill example

1. From Kali: Run a noisy technique against a DC, for example a Kerberoast attempt or a password spray.
2. In Kibana: Find the corresponding events, failed logons, ticket requests, Sysmon process creations.
3. Note which fields distinguish the attack from normal traffic, that is the actual payoff of this whole setup.

---
