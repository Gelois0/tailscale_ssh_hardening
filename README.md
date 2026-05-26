# Secure Remote Access with Tailscale and SSH Hardening

> A practical guide for building a private mesh network with Tailscale and securing SSH access with key-based authentication, firewall rules, Fail2ban, and optional port knocking.

**Technical review date:** 2026-05-26  
**Recommended level:** technical beginner / intermediate  
**Target systems:** Linux, Windows, macOS, Android, and iOS

---

## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Recommended Architecture](#2-recommended-architecture)
- [3. Prerequisites](#3-prerequisites)
- [4. Key Concepts](#4-key-concepts)
  - [4.1 Tailnet](#41-tailnet)
  - [4.2 Tailscale IP Address `100.x.y.z`](#42-tailscale-ip-address-100xyz)
  - [4.3 MagicDNS](#43-magicdns)
  - [4.4 Subnet Router](#44-subnet-router)
  - [4.5 Exit Node](#45-exit-node)
  - [4.6 Tailscale SSH](#46-tailscale-ssh)
- [5. Install Tailscale](#5-install-tailscale)
  - [5.1 Linux](#51-linux)
  - [5.2 Windows and macOS](#52-windows-and-macos)
  - [5.3 Android and iOS](#53-android-and-ios)
- [6. Authenticate Devices](#6-authenticate-devices)
- [7. Verify Connectivity](#7-verify-connectivity)
- [8. Connect with SSH over Tailscale](#8-connect-with-ssh-over-tailscale)
- [9. Use MagicDNS for Hostnames](#9-use-magicdns-for-hostnames)
- [10. Recommended SSH Hardening](#10-recommended-ssh-hardening)
  - [10.1 Create an SSH Key](#101-create-an-ssh-key)
  - [10.2 Copy the Key to the Server](#102-copy-the-key-to-the-server)
  - [10.3 Test Key-Based Login](#103-test-key-based-login)
  - [10.4 Disable Password-Based Login](#104-disable-password-based-login)
  - [10.5 Validate and Restart SSH](#105-validate-and-restart-ssh)
- [11. Restrict SSH to Tailscale with a Firewall](#11-restrict-ssh-to-tailscale-with-a-firewall)
- [12. Add Fail2ban as an Extra Layer](#12-add-fail2ban-as-an-extra-layer)
- [13. Access Controls: Grants and ACLs](#13-access-controls-grants-and-acls)
- [14. Configure a Subnet Router](#14-configure-a-subnet-router)
- [15. Configure an Exit Node](#15-configure-an-exit-node)
- [16. Optional Port Knocking with `knockd`](#16-optional-port-knocking-with-knockd)
- [17. Operational Best Practices](#17-operational-best-practices)
- [18. Troubleshooting](#18-troubleshooting)
- [19. Final Checklist](#19-final-checklist)
- [20. Official References](#20-official-references)

---

## 1. Purpose

This guide explains how to create a private network between multiple devices using **Tailscale** and how to secure remote server access over **SSH**.

The main objective is to avoid exposing SSH directly to the public Internet. Instead, Tailscale should be used as a private mesh network for remote administration.

Recommended approach:

1. Install Tailscale on all devices.
2. Authenticate every device into the same tailnet.
3. Connect by Tailscale IP or MagicDNS hostname.
4. Use SSH key-based authentication.
5. Disable SSH password login.
6. Restrict SSH to the Tailscale interface.
7. Add Fail2ban as a defensive layer.
8. Use port knocking only when a public-facing SSH port is truly required.

---

## 2. Recommended Architecture

```text
Laptop / Desktop
    |
    | Private Tailscale network
    |
Linux Server ---- tailscale0 ---- SSH only over Tailscale
    |
    └── Internal services: Docker, NAS, dashboards, databases, etc.
```

Recommended security model:

```text
Public Internet
    ↓
Firewall blocks public SSH
    ↓
Tailscale provides private network access
    ↓
SSH uses keys only, with password login disabled
```

> [!IMPORTANT]
> The safest and simplest setup is to avoid exposing SSH to the public Internet. If all your trusted devices use Tailscale, you can administer servers through the `tailscale0` interface without opening port `22` publicly.

---

## 3. Prerequisites

Before starting, make sure you have:

- A Tailscale account.
- Administrative access to the Linux machine.
- A non-root user for server administration.
- OpenSSH installed on the server.
- Internet access to install packages.
- Access to the Tailscale admin console.

Check whether SSH is installed and running.

On Debian/Ubuntu:

```bash
systemctl status ssh
```

On some distributions, the service is named `sshd`:

```bash
systemctl status sshd
```

---

## 4. Key Concepts

### 4.1 Tailnet

A **tailnet** is your private Tailscale network. Every device authenticated under the same account or organization becomes part of that private network.

Examples of devices inside a tailnet:

- Personal laptop.
- VPS server.
- Desktop workstation.
- Android or iOS phone.
- NAS.
- Raspberry Pi.

---

### 4.2 Tailscale IP Address `100.x.y.z`

Each device receives a private Tailscale IP address from the `100.64.0.0/10` range.

Example:

```text
100.101.25.8    laptop-luis
100.87.13.44    vps-server
100.92.10.20    phone
```

These are not regular public IP addresses. They are reachable inside your tailnet and allow your devices to communicate as if they were on the same private network.

Show the current device's Tailscale IPv4 address:

```bash
tailscale ip -4
```

Show both IPv4 and IPv6 Tailscale addresses:

```bash
tailscale ip
```

---

### 4.3 MagicDNS

**MagicDNS** lets you connect to devices using names instead of IP addresses.

Example:

```bash
ssh user@vps-server
```

Instead of:

```bash
ssh user@100.87.13.44
```

MagicDNS can be managed from the DNS section of the Tailscale admin console.

---

### 4.4 Subnet Router

A **Subnet Router** lets devices in your tailnet access an entire LAN behind a specific Tailscale device.

Example:

```text
Remote laptop → Tailscale → Home server → LAN 192.168.1.0/24
```

Common use cases:

- Printers.
- NAS devices.
- Internal cameras.
- Routers.
- Local-only services.

> [!NOTE]
> A Subnet Router does not route all Internet traffic. It only advertises one or more specific private subnets.

---

### 4.5 Exit Node

An **Exit Node** lets a device route all of its Internet traffic through another device in the tailnet.

Example:

```text
Laptop on public Wi-Fi → Tailscale → Home PC → Internet
```

Common use cases:

- Using a trusted connection while on public Wi-Fi.
- Browsing through your home or office network.
- Sending general Internet traffic through a specific location.

> [!WARNING]
> A Subnet Router and an Exit Node are not the same thing.
>
> A Subnet Router provides access to specific internal networks.
>
> An Exit Node routes general Internet traffic through another Tailscale device.

---

### 4.6 Tailscale SSH

**Tailscale SSH** lets Tailscale manage SSH access based on tailnet identity and access policies.

It is not mandatory. You can either:

- Use traditional OpenSSH with SSH keys.
- Use Tailscale SSH for policy-managed SSH access inside the tailnet.

Enable Tailscale SSH on a destination machine:

```bash
sudo tailscale set --ssh
```

Disable Tailscale SSH:

```bash
sudo tailscale set --ssh=false
```

> [!NOTE]
> If you prefer a universal and distribution-neutral setup, use OpenSSH with SSH keys and restrict access to the Tailscale interface.

---

## 5. Install Tailscale

### 5.1 Linux

On mainstream Linux distributions:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Then start authentication:

```bash
sudo tailscale up
```

The command prints an authentication URL. Open it in a browser and sign in with the account you want to use for your tailnet.

After authentication, verify the device appears in the Tailscale admin console.

---

### 5.2 Windows and macOS

1. Download the installer from the official Tailscale download page.
2. Install the application.
3. Sign in with the same account used for your tailnet.
4. Confirm the device appears in the Tailscale admin console.

---

### 5.3 Android and iOS

1. Install Tailscale from Google Play or the App Store.
2. Sign in with the same account.
3. Enable the Tailscale connection.
4. Confirm the device appears as active in the admin console.

---

## 6. Authenticate Devices

On Linux:

```bash
sudo tailscale up
```

If the device was already authenticated and you need to force re-authentication:

```bash
sudo tailscale up --force-reauth
```

For servers that must remain connected continuously, review device key expiry in the Tailscale admin console.

> [!WARNING]
> Disabling key expiry can be useful for trusted servers, but it reduces security if the machine is lost, reused, or compromised. Revoke unused devices immediately.

---

## 7. Verify Connectivity

Show the tailnet status:

```bash
tailscale status
```

Example output:

```text
100.87.13.44    vps-server       user@   linux   active
100.101.25.8    laptop-luis      user@   linux   active
100.92.10.20    iphone-luis      user@   ios     active
```

Show the current device's Tailscale IP:

```bash
tailscale ip -4
```

Test connectivity to another device:

```bash
ping 100.87.13.44
```

---

## 8. Connect with SSH over Tailscale

Connect using the Tailscale IP:

```bash
ssh user@100.87.13.44
```

Connect using MagicDNS:

```bash
ssh user@vps-server
```

Example:

```bash
ssh gelois@100.87.13.44
```

> [!TIP]
> If SSH works over Tailscale, you usually do not need to expose port `22` to the public Internet.

---

## 9. Use MagicDNS for Hostnames

MagicDNS avoids depending on numeric IP addresses.

Example:

```bash
ssh user@my-server
```

Recommendations:

- Use clear machine names.
- Avoid generic names such as `server`, `test`, or `desktop`.
- Prefer names such as `vps-cuenca`, `home-nas`, or `work-laptop`.

---

## 10. Recommended SSH Hardening

### 10.1 Create an SSH Key

On your client machine:

```bash
ssh-keygen -t ed25519 -C "user@machine"
```

Recommended default path:

```text
~/.ssh/id_ed25519
```

---

### 10.2 Copy the Key to the Server

Using the Tailscale IP:

```bash
ssh-copy-id user@100.87.13.44
```

Using MagicDNS:

```bash
ssh-copy-id user@vps-server
```

---

### 10.3 Test Key-Based Login

Before disabling password authentication, open a new terminal and test login with the key:

```bash
ssh user@100.87.13.44
```

> [!IMPORTANT]
> Do not close your existing SSH session until you have confirmed that a new session works with the SSH key.

---

### 10.4 Disable Password-Based Login

Edit the SSH daemon configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure these values are set:

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Recommended extra hardening:

```sshconfig
X11Forwarding no
```

> [!NOTE]
> Some cloud images or distributions place overrides inside `/etc/ssh/sshd_config.d/`. If password authentication still appears enabled after editing `sshd_config`, inspect that directory as well.

Check the effective SSH configuration:

```bash
sudo sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication'
```

---

### 10.5 Validate and Restart SSH

Validate the configuration before restarting:

```bash
sudo sshd -t
```

If no errors are shown, restart SSH.

On Debian/Ubuntu:

```bash
sudo systemctl restart ssh
```

On RHEL, Fedora, Arch, and many other distributions:

```bash
sudo systemctl restart sshd
```

Check service status:

```bash
systemctl status ssh
```

Or:

```bash
systemctl status sshd
```

---

## 11. Restrict SSH to Tailscale with a Firewall

If you use UFW, allow SSH only through the `tailscale0` interface.

Allow SSH from Tailscale:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

Deny public SSH:

```bash
sudo ufw deny 22/tcp
```

Enable UFW if it is not already enabled:

```bash
sudo ufw enable
```

Show numbered firewall rules:

```bash
sudo ufw status numbered
```

> [!WARNING]
> Before changing firewall rules on a remote server, keep an existing SSH session open and test a second connection through Tailscale.

---

## 12. Add Fail2ban as an Extra Layer

Fail2ban detects repeated failed login attempts and can temporarily ban source IP addresses.

> [!NOTE]
> Fail2ban is an additional defense layer. It does not replace SSH keys, disabled password login, or firewall restrictions.

Install on Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y fail2ban
```

Create a local SSH jail:

```bash
sudo nano /etc/fail2ban/jail.d/sshd.local
```

Recommended baseline configuration:

```ini
[sshd]
enabled = true
backend = systemd
port = ssh
filter = sshd
maxretry = 5
findtime = 10m
bantime = 1h
```

Restart Fail2ban:

```bash
sudo systemctl restart fail2ban
```

Show general status:

```bash
sudo fail2ban-client status
```

Show SSH jail status:

```bash
sudo fail2ban-client status sshd
```

---

## 13. Access Controls: Grants and ACLs

Tailscale access controls define which users, groups, devices, or tagged machines can communicate with each other.

Common uses:

- Allow only administrators to reach servers.
- Separate personal devices from infrastructure.
- Limit access by port.
- Control Tailscale SSH access.

Conceptual grants example:

```json
{
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:servers"],
      "ip": ["tcp:22"]
    }
  ]
}
```

> [!NOTE]
> Routes define where traffic can go. Access controls define whether that traffic is allowed.

---

## 14. Configure a Subnet Router

A Subnet Router allows your tailnet devices to access a full LAN behind one Tailscale machine.

Example: expose `192.168.1.0/24` to the tailnet.

Enable IP forwarding on Linux:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise the subnet route:

```bash
sudo tailscale set --advertise-routes=192.168.1.0/24
```

Then approve the route in the Tailscale admin console.

Verify status:

```bash
tailscale status
```

> [!TIP]
> If you want to route all Internet traffic, use an Exit Node instead of advertising default routes as a subnet route.

---

## 15. Configure an Exit Node

An Exit Node routes a client's general Internet traffic through another device in the tailnet.

Enable IP forwarding on Linux:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise the current machine as an Exit Node:

```bash
sudo tailscale set --advertise-exit-node
```

Then approve the Exit Node in the Tailscale admin console.

Use an Exit Node from Linux:

```bash
sudo tailscale set --exit-node=exit-node-name
```

Stop using an Exit Node:

```bash
sudo tailscale set --exit-node=
```

> [!NOTE]
> Each client decides whether to use an Exit Node. It is not automatically applied to every device in the tailnet.

---

## 16. Optional Port Knocking with `knockd`

Port knocking keeps a port closed until a client sends a predefined sequence of connection attempts to specific ports.

Conceptual flow:

```text
Client knocks: 7000 → 8000 → 9000
Server temporarily opens SSH for that source IP
Client connects over SSH
Server closes SSH access again
```

### When It Makes Sense

Port knocking may be useful when:

- SSH must be exposed to the public Internet.
- You want to hide a port from basic scans.
- You cannot restrict SSH to a fixed source IP.

### When It Is Not Needed

Port knocking is usually unnecessary when:

- SSH is only reachable over Tailscale.
- Port `22` is not exposed publicly.
- Access is already controlled by firewall rules, SSH keys, and Tailscale policies.

Install on Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y knockd
```

Enable `knockd`:

```bash
sudo nano /etc/default/knockd
```

Set:

```ini
START_KNOCKD=1
KNOCKD_OPTS="-i eth0"
```

> [!WARNING]
> Replace `eth0` with the actual public interface of your server. Check interfaces with `ip addr`.

Edit the knockd configuration:

```bash
sudo nano /etc/knockd.conf
```

Basic example using UFW:

```ini
[options]
UseSyslog

[openSSH]
sequence = 7000,8000,9000
seq_timeout = 15
tcpflags = syn
command = /usr/sbin/ufw allow from %IP% to any port 22 proto tcp
cmd_timeout = 30
stop_command = /usr/sbin/ufw delete allow from %IP% to any port 22 proto tcp
```

Restart knockd:

```bash
sudo systemctl restart knockd
```

From the client:

```bash
knock SERVER_PUBLIC_IP 7000 8000 9000
ssh user@SERVER_PUBLIC_IP
```

> [!CAUTION]
> Port knocking can lock you out if the interface, sequence, firewall command, or timeout is wrong. Test it in a controlled environment before relying on it on a production server.

---

## 17. Operational Best Practices

- Keep Tailscale updated.
- Use non-root users for administration.
- Prefer Ed25519 SSH keys.
- Disable SSH password login.
- Disable root login over SSH.
- Do not expose SSH publicly when Tailscale is available.
- Restrict internal services with firewall rules.
- Use MagicDNS to reduce mistakes with IP addresses.
- Review authorized devices in the Tailscale admin console.
- Revoke devices that are no longer used.
- Use tags and access controls for multi-server environments.
- Document firewall and SSH changes before applying them.

---

## 18. Troubleshooting

### The Device Does Not Appear in Tailscale

Check status:

```bash
tailscale status
```

Force re-authentication:

```bash
sudo tailscale up --force-reauth
```

---

### I Do Not Know My Tailscale IP

```bash
tailscale ip -4
```

---

### SSH Does Not Work

Check connectivity:

```bash
ping 100.x.y.z
```

Run SSH in verbose mode:

```bash
ssh -vvv user@100.x.y.z
```

Check the SSH service on the server:

```bash
sudo systemctl status ssh
```

Or:

```bash
sudo systemctl status sshd
```

---

### I Changed SSH and Now I Cannot Log In

If you still have an open session, validate the configuration:

```bash
sudo sshd -t
```

Review logs:

```bash
sudo journalctl -u ssh -n 100 --no-pager
```

Or:

```bash
sudo journalctl -u sshd -n 100 --no-pager
```

---

### Fail2ban Is Not Detecting SSH Attempts

Check the SSH jail:

```bash
sudo fail2ban-client status sshd
```

Check Fail2ban logs:

```bash
sudo journalctl -u fail2ban -n 100 --no-pager
```

---

## 19. Final Checklist

- [ ] Tailscale is installed on all devices.
- [ ] All devices are authenticated into the same tailnet.
- [ ] `tailscale status` shows active devices.
- [ ] `tailscale ip -4` returns a `100.x.y.z` address.
- [ ] MagicDNS is enabled.
- [ ] SSH works through the Tailscale IP.
- [ ] SSH works through MagicDNS.
- [ ] An Ed25519 SSH key has been created.
- [ ] The SSH key has been copied to the server.
- [ ] Key-based login has been tested from a second terminal.
- [ ] `PasswordAuthentication no` is configured.
- [ ] `KbdInteractiveAuthentication no` is configured.
- [ ] `PermitRootLogin no` is configured.
- [ ] SSH configuration has been validated with `sudo sshd -t`.
- [ ] SSH has been restarted successfully.
- [ ] The firewall allows SSH through `tailscale0`.
- [ ] Public SSH is blocked unless strictly required.
- [ ] Fail2ban is installed and running.
- [ ] The `sshd` jail is active.
- [ ] A Subnet Router is configured only if LAN access is required.
- [ ] An Exit Node is configured only if Internet routing is required.
- [ ] Port knocking is configured only if it is truly needed.
- [ ] Old or unused devices have been revoked from the tailnet.

---

## 20. Official References

- [Tailscale: Install on Linux](https://tailscale.com/docs/install/linux)
- [Tailscale: Download and install](https://tailscale.com/docs/install)
- [Tailscale: MagicDNS](https://tailscale.com/docs/features/magicdns)
- [Tailscale: Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh)
- [Tailscale: Subnet Routers](https://tailscale.com/docs/features/subnet-routers)
- [Tailscale: Exit Nodes](https://tailscale.com/docs/features/exit-nodes)
- [Tailscale: Access Controls](https://tailscale.com/docs/features/access-control/acls)
- [OpenSSH: sshd_config manual](https://man.openbsd.org/sshd_config)
- [Fail2ban: Official project](https://github.com/fail2ban/fail2ban)
- [Ubuntu manpage: knockd](https://manpages.ubuntu.com/manpages/focal/man1/knockd.1.html)

---

## Editorial Note for GitHub and HackMD

This document uses standard Markdown with hierarchical headings, code blocks, callouts, and a manual table of contents.

For HackMD, you can add this near the top if you want automatic navigation:

```markdown
[TOC]
```

For GitHub, the manual table of contents included in this document is usually cleaner.
