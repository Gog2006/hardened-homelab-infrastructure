System: Ubuntu Server 22.04 LTS  
Environment: Proxmox VM  
Purpose: Secure baseline configuration (SSH, Firewall, Updates)

---

## Variables (Adapt Once)

```bash
USER=<your_user>
HOST=<vm_ip_or_hostname>
KEY_PATH=~/.ssh/app-vm
```

---

## 1. Generate SSH Key (Windows PowerShell)

```bash
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\app-vm" -C "app-vm"
```

Explanation: Generates a secure SSH key pair (private + public).

---

## 2. Initial SSH Login (Password)

```bash
ssh $USER@$HOST
```

Explanation: First connection using password authentication.

---

## 3. Prepare SSH Directory (on VM)

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Explanation:
- Create `.ssh` directory
- Secure permissions
- Prepare authorized keys file

---

## 4. Add Public Key

### Option A (Recommended - from Windows)

```bash
type $env:USERPROFILE\.ssh\app-vm.pub | ssh $USER@$HOST "cat >> ~/.ssh/authorized_keys"
```

Explanation: Securely copies your public key to the VM.

### Verify

```bash
cat ~/.ssh/authorized_keys
```

Explanation: Confirms the key is installed.

---

## 5. Install Nano (Optional)

```bash
sudo apt install nano -y
```

Explanation: Installs a simple text editor.

---

## 6. Secure SSH Configuration

```bash
sudo sed -i 's/#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sudo sed -i 's/#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
```

Explanation:
- Disable password authentication
- Enable key-based authentication
- Disable root login

### Verify

```bash
grep -E "PasswordAuthentication|PubkeyAuthentication|PermitRootLogin" /etc/ssh/sshd_config
```

### Apply Changes

```bash
sudo systemctl restart sshd
```

---

## 7. Configure Firewall (UFW)

### Default Policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### Allow Required Ports

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 9121/tcp
```

### Block Redis External Access

```bash
sudo ufw deny 6379/tcp
```

### Enable Firewall

```bash
sudo ufw enable
```

### Verify

```bash
sudo ufw status
```

---

## 8. System Updates

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo apt list --upgradable
```

Explanation:
- Update system packages
- Remove unused dependencies
- Confirm system is fully updated

---

## 9. Test SSH Key Login

```bash
ssh -i "$env:USERPROFILE\.ssh\app-vm" $USER@$HOST
```

Explanation: Connect using SSH key (passwordless).

---

## 10. Optional SSH Client Configuration

Create file:

```bash
~/.ssh/config
```

Add:

```bash
Host app-vm
    HostName <HOST>
    User <USER>
    IdentityFile ~/.ssh/app-vm
    StrictHostKeyChecking no
```

Usage:

```bash
ssh app-vm
```

---

## Reusability

To reuse this setup on another VM:

```bash
USER=newuser
HOST=192.168.x.x
KEY_PATH=~/.ssh/new-key
```

Then repeat the same steps.

---

## 
## Troubleshooting

### SSH Access Failure

```bash
ssh -i "$env:USERPROFILE\.ssh\app-vm" $USER@$HOST
```

Check:
- ~/.ssh/authorized_keys
- SSH config

---

### Firewall Issues

```bash
sudo ufw status
sudo ufw allow <PORT>/tcp
sudo ufw reload
```

---

### Permission Fix

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

---


