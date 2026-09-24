
# Ubuntu Server Hardening: SSH Key Authentication + UFW Firewall

Hardening a fresh Ubuntu 26.04 LTS server: key-only SSH access, root login disabled, and a default-deny firewall.

## Objective
Reduce the attack surface of a Linux server by:
- Replacing password login with SSH key authentication
- Disabling direct root login over SSH
- Blocking all incoming traffic except SSH with UFW

## Environment
- Ubuntu 26.04.1 LTS (VMware Workstation virtual machine)
- OpenSSH server 10.x
- UFW 0.36
- Client: Windows 10 with the built-in OpenSSH client (PowerShell)

## Steps

### 0. Safety net
Took a VM snapshot before changing any SSH settings, so a lockout could be reverted.

### 1. Install and enable SSH
```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
ss -tlnp | grep :22
```

### 2. Generate a key pair on the client (PowerShell) and install the public key
```powershell
ssh-keygen -t ed25519 -C "elwany-vm"
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@<VM_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh <user>@<VM_IP>     # confirm key login works BEFORE disabling passwords
```

### 3. Harden sshd
Created `/etc/ssh/sshd_config.d/00-hardening.conf` (see [`configs/00-hardening.conf`](configs/00-hardening.conf)):
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```
The `00-` prefix makes sure it is read first, because sshd uses the first value it finds for each option.

```bash
sudo sshd -t                                   # syntax check, no output = OK
sudo sshd -T | grep -Ei 'rootlogin|passwordauth|pubkeyauth'   # effective values
sudo systemctl restart ssh
```

### 4. Firewall with UFW
The order matters: allow SSH **before** enabling the firewall.
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw --force enable
sudo ufw status verbose
```

## Verification
| Test | Expected | Result |
|---|---|---|
| SSH with key | Login succeeds | Passed |
| SSH with password only (`-o PubkeyAuthentication=no`) | `Permission denied (publickey)` | Passed |
| SSH as root | `Permission denied (publickey)` | Passed |
| `ufw status verbose` | active, deny incoming / allow outgoing, 22/tcp allowed | Passed |
| SSH login after enabling UFW | Login succeeds | Passed |

## Screenshots
See the [`screenshots/`](screenshots/) folder:
1. `01-ssh-key-login.png` — successful key login
2. `02-sshd-effective-config.png` — `sshd -T` output
3. `03-password-denied.png` — password login refused
4. `04-root-denied.png` — root login refused
5. `05-ufw-status.png` — firewall status

## Notes and lessons learned
- Test the new key login from a **second** session before disabling passwords, and keep the first session open.
- Never run `ufw enable` before `ufw allow OpenSSH`, or you lock yourself out.
- On Ubuntu 24.04+ the SSH service can be socket-activated (`ssh.socket`), so it may show `inactive` until the first connection; `systemctl enable --now ssh` makes it explicit.
- Docker-published ports bypass UFW rules (Docker manages its own iptables rules), so UFW alone does not protect container ports.
- Do **not** commit private keys or `authorized_keys` to this repository.
- Environment: local virtual machine, for learning and demonstration.

## Next
Automate this exact setup with an Ansible playbook.
