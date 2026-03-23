# WireGuard VPN Server Setup Guide

A complete, step-by-step guide to creating a secure, private VPN using WireGuard on a cloud server.

**Setup Time**: ~45 minutes  
**Monthly Cost**: ~$5 USD  
**Difficulty**: Beginner-Intermediate

---

## Table of Contents
- [What is WireGuard?](#what-is-wireguard)
- [Prerequisites](#prerequisites)
- [Part 1: Server Setup & Manual Configuration](#part-1-server-setup--manual-configuration)
- [Part 2: Client Configuration](#part-2-client-configuration)
- [Part 3: Connect Client & Enable Routing](#part-3-connect-client--enable-routing)
- [Part 4: Make Configuration Persistent](#part-4-make-configuration-persistent)
- [Part 5: Firewall Configuration](#part-5-firewall-configuration)
- [Part 6: SSH Hardening](#part-6-ssh-hardening)
- [Part 7: Testing & Verification](#part-7-testing--verification)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

---

## What is WireGuard?

WireGuard is a modern, open-source VPN protocol known for its **speed**, **simplicity**, and **strong security**.

**Key Advantages:**
- **Minimal codebase** (~4,000 lines) - easy to audit and secure
- **State-of-the-art cryptography** (ChaCha20, Curve25519)
- **Significantly faster** than OpenVPN or IPSec
- **Simpler configuration** - no complex certificate management
- **Smaller attack surface** - less code means fewer vulnerabilities

This guide will help you deploy your own WireGuard server on a cheap cloud VPS, giving you full control over your privacy.

---

## Prerequisites

Before starting, ensure you have:

- **Basic Linux command-line knowledge** (navigating directories, editing files)
- **A credit card** for VPS provider signup (most offer free credits)
- **WireGuard client app** for your device:
  - Windows: [Download from wireguard.com](https://www.wireguard.com/install/)
  - macOS: Install via App Store or Homebrew
  - Linux: `sudo apt install wireguard` (or equivalent)
  - iOS/Android: Search "WireGuard" in app store
- **SSH client**:
  - Mac/Linux: Built-in Terminal
  - Windows: PowerShell, Command Prompt, or [PuTTY](https://www.putty.org/)

---

## Part 1: Server Setup & Manual Configuration

### Step 1: Create Your Cloud Server

We'll use Akamai (formerly Linode) for this guide, but these steps work on any Ubuntu-based VPS (DigitalOcean, Vultr, AWS Lightsail, etc.).

**Akamai/Linode Setup:**
1. Create an account at [linode.com](https://www.linode.com/) (get $100 free credit)
2. Log in and click **Create Linode** (top-right)
3. Configure your server:
   - **Region**: Choose a location close to you for best speeds (or a different country for geo-unblocking)
   - **Distribution**: Select **Ubuntu 24.04 LTS** (or 22.04 LTS)
   - **Plan**: Choose **Shared CPU - Nanode 1GB** (~$5/month)
     - WireGuard is extremely lightweight; this is more than enough
   - **Label**: Name it something like `MyWireGuardVPN`
   - **Root Password**: Create a **strong, unique password** (save it securely)
4. Click **Create Linode**

Wait 1-2 minutes for the server to boot. Once it shows **Running**, copy the **IP Address** (e.g., `203.0.113.45`).

---

### Step 2: Connect and Update Your Server

**Connect via SSH:**

Open your terminal and connect to the server:

```bash
ssh root@[SERVER_IP]
```

Replace `[SERVER_IP]` with your server's IP address (e.g., `ssh root@203.0.113.45`).

- Type `yes` when asked to trust the host
- Enter the root password you created in Step 1

**Update the system:**

```bash
# Update package lists
sudo apt update

# Upgrade all packages to latest versions
sudo apt upgrade -y
```

This may take 2-5 minutes. Reboot if prompted.

---

### Step 3: Install WireGuard

Install the WireGuard package:

```bash
sudo apt install wireguard -y
```

**Expected output:**
```
Reading package lists... Done
Building dependency tree... Done
wireguard is already the newest version (1.0.20210914-1ubuntu2)
```

---

### Step 4: Generate Server Keys

WireGuard uses public-key cryptography (like SSH). Each device needs a private/public key pair.

**Generate the server's keys:**

```bash
# Navigate to WireGuard directory
cd /etc/wireguard

# Generate private key
wg genkey | sudo tee server_privatekey

# Set restrictive permissions (critical!)
sudo chmod 600 server_privatekey

# Generate public key from private key
sudo cat server_privatekey | wg pubkey | sudo tee server_publickey
```

**View and save your keys:**

```bash
# Display private key (keep this SECRET!)
sudo cat server_privatekey

# Display public key (you'll share this with clients)
sudo cat server_publickey
```

**Copy both keys** to a temporary text file on your local computer. You'll need:
- **Server private key** - for server configuration (Part 1, Step 5)
- **Server public key** - for client configuration (Part 2)

**Example output:**
```
Private key: 4K3JxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxGE=
Public key:  7H9LxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxwM=
```

⚠️ **Security Note**: Never share your private key with anyone!

---

### Step 5: Manually Build the WireGuard Interface

These commands create a temporary WireGuard interface. We'll make it persistent in Part 4.

```bash
# 1. Create the virtual network interface named 'wg0'
sudo ip link add wg0 type wireguard

# 2. Assign a private IP address to this interface
sudo ip addr add 10.0.0.1/24 dev wg0

# 3. Link your server's private key to the interface
sudo wg set wg0 private-key /etc/wireguard/server_privatekey

# 4. Set the listening port (default: 51820)
sudo wg set wg0 listen-port 51820

# 5. Bring the interface online
sudo ip link set wg0 up
```

**What does `10.0.0.1/24` mean?**
- This is a private subnet for your VPN
- `10.0.0.1` = Server's VPN IP address
- `/24` = Subnet mask (supports 254 devices: 10.0.0.2 - 10.0.0.254)

---

### Step 6: Verify the Server Status

Check that WireGuard is running:

```bash
sudo wg show wg0
```

**Expected output:**
```
interface: wg0
  public key: 7H9Lxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
  private key: (hidden)
  listening port: 51820
```

**Save the listening port** (usually `51820`) - you'll need it for client configuration.

📌 **Note**: At this point, the interface is running but will disappear after a reboot. We'll fix this in Part 4.

---

## Part 2: Client Configuration

Now, configure your client device (laptop, phone, etc.) to connect to the VPN.

### Step 1: Install WireGuard Client

Download and install the WireGuard app for your OS (see [Prerequisites](#prerequisites)).

---

### Step 2: Generate Client Keys

**Option A: Let the app generate keys (Recommended for beginners)**
- Open the WireGuard app
- Click "Add Tunnel" → "Add Empty Tunnel"
- The app will auto-generate a private key

**Option B: Generate keys manually (Linux/macOS/advanced users)**

```bash
# On your local machine
wg genkey | tee client_privatekey | wg pubkey > client_publickey

# View the keys
cat client_privatekey
cat client_publickey
```

---

### Step 3: Create Client Configuration File

Create a new tunnel configuration in the WireGuard app with the following content:

```ini
[Interface]
PrivateKey = [CLIENT_PRIVATE_KEY]
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = [SERVER_IP]:51820
PersistentKeepalive = 25
```

**Replace the placeholders:**
- `[CLIENT_PRIVATE_KEY]` - Client's private key (generated in Step 2)
- `[SERVER_PUBLIC_KEY]` - Server's public key (from Part 1, Step 4)
- `[SERVER_IP]` - Your server's public IP address (e.g., `203.0.113.45`)

**Configuration Explained:**
- **Address**: Client's VPN IP address (`10.0.0.2`)
- **DNS**: Cloudflare DNS (alternatives: `8.8.8.8` for Google, `9.9.9.9` for Quad9)
- **AllowedIPs**: Route all traffic through VPN (`0.0.0.0/0` = IPv4, `::/0` = IPv6)
- **Endpoint**: Server's public IP and WireGuard port
- **PersistentKeepalive**: Keeps connection alive through NAT (25 seconds)

**Copy the client's public key** - you'll need it in Part 3 to authorize this device on the server.

---

### Step 4: Prevent IPv6 Leaks (Optional but Recommended)

To prevent IP leaks, disable IPv6 on your client machine.

**Windows:**
1. Go to: `Control Panel` → `Network and Internet` → `Network and Sharing Center`
2. Click your network adapter → `Properties`
3. Uncheck **Internet Protocol Version 6 (TCP/IPv6)**
4. Click OK

**macOS:**
1. Go to: `System Settings` → `Network`
2. Select your active connection → `Details` → `TCP/IP`
3. Set IPv6 to **Link-local only** or **Off**

**Linux:**
```bash
# Temporarily disable IPv6
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1

# Make it permanent
echo "net.ipv6.conf.all.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
```

**Mobile (iOS/Android):**  
The WireGuard app handles this automatically when `AllowedIPs = 0.0.0.0/0, ::/0` is configured.

---

## Part 3: Connect Client & Enable Routing

### Step 1: Add Client as an Authorized Peer

Back on the **server** (via SSH), authorize your client to connect:

```bash
sudo wg set wg0 peer [CLIENT_PUBLIC_KEY] allowed-ips 10.0.0.2/32
```

Replace `[CLIENT_PUBLIC_KEY]` with the public key from your client configuration (Part 2, Step 3).

**Verify the peer was added:**

```bash
sudo wg show wg0
```

**Expected output:**
```
interface: wg0
  public key: 7H9Lxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
  private key: (hidden)
  listening port: 51820

peer: 9K2Mxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
  allowed ips: 10.0.0.2/32
```

---

### Step 2: Enable IPv4 Forwarding (NAT)

This allows the VPN server to route your traffic to the internet.

```bash
# 1. Enable IP forwarding
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# 2. Apply changes immediately
sudo sysctl -p

# 3. Verify forwarding is enabled (should return: 1)
cat /proc/sys/net/ipv4/ip_forward
```

---

### Step 3: Identify Your Network Interface

Find your server's public network interface name:

```bash
ip route | grep default
```

**Expected output:**
```
default via 203.0.113.1 dev eth0 proto static
```

The interface name is after `dev` - common names include:
- `eth0` (traditional Ethernet)
- `ens3`, `enp0s3` (predictable network names)
- `eno1` (onboard Ethernet)

**Copy the interface name** (e.g., `eth0`) for the next step.

---

### Step 4: Configure NAT (Masquerading)

This "hides" your client's VPN IP behind the server's public IP.

```bash
# Replace 'eth0' with YOUR interface name from Step 3
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**Verify the NAT rule:**

```bash
sudo iptables -t nat -L POSTROUTING -v -n
```

**Expected output:**
```
Chain POSTROUTING (policy ACCEPT)
target     prot opt source      destination
MASQUERADE all  --  0.0.0.0/0   0.0.0.0/0
```

---

### Step 5: Test the Connection

On your **client device**:
1. Open the WireGuard app
2. **Activate** the VPN tunnel
3. Check the status - it should show "Active" with data transfer

If connected successfully, proceed to Part 4 to make this configuration permanent.

---

## Part 4: Make Configuration Persistent

Right now, your VPN works but will **reset after a reboot**. Let's fix that.

### Step 1: Save IPTables Rules

Install the persistence package:

```bash
sudo apt install iptables-persistent -y
```

When prompted:
- **Save current IPv4 rules?** → Yes
- **Save current IPv6 rules?** → Yes

**Make sure rules persist across reboots:**

```bash
sudo netfilter-persistent save
```

---

### Step 2: Create WireGuard Configuration File

Save your live configuration to a file:

```bash
# Save current WireGuard configuration
sudo wg showconf wg0 | sudo tee /etc/wireguard/wg0.conf
```

**Verify the configuration:**

```bash
sudo cat /etc/wireguard/wg0.conf
```

**Expected output:**
```ini
[Interface]
ListenPort = 51820
PrivateKey = 4K3JxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxGE=

[Peer]
PublicKey = 9K2Mxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
AllowedIPs = 10.0.0.2/32
```

**Set secure permissions:**

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

---

### Step 3: Enable WireGuard as a System Service

This starts WireGuard automatically on boot:

```bash
# Enable auto-start on boot
sudo systemctl enable wg-quick@wg0

# Stop the manual interface (from Part 1)
sudo ip link del wg0

# Start the systemd service version
sudo systemctl start wg-quick@wg0

# Check status
sudo systemctl status wg-quick@wg0
```

**Expected output:**
```
● wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0
     Loaded: loaded
     Active: active (exited) since [timestamp]
```

**Test persistence:**

```bash
# Reboot the server
sudo reboot

# Wait 30 seconds, then reconnect
ssh root@[SERVER_IP]

# Check if WireGuard started automatically
sudo wg show
```

If you see your interface and peer, persistence is working! ✅

---

## Part 5: Firewall Configuration

Secure your server with UFW (Uncomplicated Firewall).

### Step 1: Install and Configure UFW

```bash
# Install UFW (usually pre-installed on Ubuntu)
sudo apt install ufw -y

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (IMPORTANT: Use port 22 for now; we'll change it in Part 6)
sudo ufw allow 22/tcp

# Allow WireGuard
sudo ufw allow 51820/udp

# Enable the firewall
sudo ufw enable
```

When prompted: **Command may disrupt existing ssh connections. Proceed with operation (y|n)?** → Type `y`

**Verify firewall status:**

```bash
sudo ufw status verbose
```

**Expected output:**
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
51820/udp                  ALLOW       Anywhere
```

⚠️ **Critical**: Make sure port 22 is allowed **before** enabling UFW, or you'll lock yourself out!

---

## Part 6: SSH Hardening

Secure SSH access to prevent unauthorized logins.

### Step 1: Create a New Sudo User

**Never use root for daily tasks.** Create a dedicated user:

```bash
# Create user with home directory and bash shell
sudo useradd -m -s /bin/bash ghostvpn

# Set a strong password
sudo passwd ghostvpn

# Add user to sudo group (admin privileges)
sudo usermod -aG sudo ghostvpn
```

---

### Step 2: Set Up SSH Key Authentication

**On your local machine** (not the server), generate an SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -C "yourname@ghostvpn"
```

Follow the prompts:
- **File location**: Press Enter (use default: `~/.ssh/id_rsa`)
- **Passphrase**: Highly recommended for extra security

**Copy the public key to the server:**

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ghostvpn@[SERVER_IP]
```

Enter the `ghostvpn` user's password when prompted.

**Test key-based login:**

```bash
ssh ghostvpn@[SERVER_IP]
```

If you can log in without entering a password (or only the key's passphrase), it's working! ✅

---

### Step 3: Lock Down SSH Configuration

**Back on the server** (logged in as `ghostvpn`):

```bash
# Backup the original config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Edit the SSH config
sudo nano /etc/ssh/sshd_config
```

**Find and modify these lines** (use `Ctrl+W` to search in nano):

```
# Change SSH port from 22 to 2222 (reduces automated attacks)
Port 2222

# Disable root login completely
PermitRootLogin no

# Enable public key authentication
PubkeyAuthentication yes

# Disable password authentication (keys only)
PasswordAuthentication no

# Ensure authorized keys file is enabled
AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
```

**Save and exit:**
- Press `Ctrl+O` (save)
- Press `Enter` (confirm)
- Press `Ctrl+X` (exit)

---

### Step 4: Update Firewall for New SSH Port

**Before restarting SSH**, allow the new port:

```bash
# Allow new SSH port
sudo ufw allow 2222/tcp

# Remove old SSH port
sudo ufw delete allow 22/tcp

# Verify the change
sudo ufw status
```

---

### Step 5: Restart SSH Service

**Ubuntu 24.04+ (socket-based SSH):**

```bash
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket
sudo systemctl enable ssh
sudo systemctl restart ssh
```

**Ubuntu 22.04 or older (traditional SSH):**

```bash
sudo systemctl restart ssh
```

**Verify SSH is listening on the new port:**

```bash
sudo ss -tulnp | grep ssh
```

**Expected output:**
```
tcp   LISTEN 0   128   0.0.0.0:2222   0.0.0.0:*   users:(("sshd",pid=1234,fd=3))
```

---

### Step 6: Test the New Configuration

**Open a NEW terminal window** (don't close your current session yet!):

```bash
ssh -p 2222 ghostvpn@[SERVER_IP]
```

If you can log in successfully:
- ✅ SSH hardening is complete
- ✅ Root login is now blocked
- ✅ Password login is now blocked

If the connection fails, **keep your old terminal window open** and review the SSH config for errors.

---

## Part 7: Testing & Verification

### Step 1: Verify VPN Connection

**On your client device**, connect to the VPN and run these tests:

**Check your public IP:**

```bash
# Before connecting VPN
curl ifconfig.me

# After connecting VPN
curl ifconfig.me
```

The IP should change to your **server's IP address**. ✅

---

### Step 2: Test VPN Connectivity

**Ping the server from inside the VPN:**

```bash
ping 10.0.0.1
```

**Expected output:**
```
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=25.3 ms
```

If you see replies, the VPN tunnel is working! ✅

---

### Step 3: Check for DNS Leaks

Visit [https://dnsleaktest.com/](https://dnsleaktest.com/) and run an extended test.

**Expected result:**
- DNS servers should belong to Cloudflare (1.1.1.1), **not your ISP**
- Geolocation should match your **server's location**

If you see your ISP's DNS, check your client config's `DNS` setting (Part 2, Step 3).

---

### Step 4: Check for IPv6 Leaks

Visit [https://ipleak.net/](https://ipleak.net/)

**Expected result:**
- **IPv4 address**: Should show your server's IP
- **IPv6 address**: Should show **nothing** (or server IPv6 if configured)

If your real IPv6 address is visible, review Part 2, Step 4.

---

### Step 5: Test Connection Persistence

**Disconnect and reconnect the VPN** several times. Each time, check:

```bash
curl ifconfig.me
```

It should consistently show the server's IP. ✅

---

## Troubleshooting

### Issue: Can't Connect to SSH After Hardening

**Symptom:** `Connection refused` or `Connection timed out` on port 2222

**Solutions:**
1. **Check if SSH is running:**
   ```bash
   sudo systemctl status ssh
   ```
   If stopped, restart it: `sudo systemctl restart ssh`

2. **Verify the firewall allows port 2222:**
   ```bash
   sudo ufw status | grep 2222
   ```
   If not listed, add it: `sudo ufw allow 2222/tcp`

3. **Check SSH is listening on the correct port:**
   ```bash
   sudo ss -tulnp | grep ssh
   ```
   Should show `0.0.0.0:2222`

4. **Access via cloud provider's console:**
   - Log in to your VPS provider's dashboard
   - Use the built-in console/terminal (bypasses SSH)
   - Fix the `/etc/ssh/sshd_config` file

---

### Issue: VPN Connects But No Internet

**Symptom:** VPN shows "Active" but websites don't load

**Solutions:**
1. **Check IP forwarding is enabled:**
   ```bash
   cat /proc/sys/net/ipv4/ip_forward
   ```
   Should return `1`. If not: `sudo sysctl -w net.ipv4.ip_forward=1`

2. **Verify NAT rule exists:**
   ```bash
   sudo iptables -t nat -L POSTROUTING -v -n
   ```
   Should show `MASQUERADE` rule. If missing, re-run Part 3, Step 4.

3. **Check correct network interface:**
   ```bash
   ip route | grep default
   ```
   Use the interface name shown in the iptables command.

4. **Restart WireGuard:**
   ```bash
   sudo systemctl restart wg-quick@wg0
   ```

---

### Issue: VPN Connection Drops Frequently

**Symptom:** Connection disconnects every few minutes

**Solutions:**
1. **Increase `PersistentKeepalive` in client config:**
   ```ini
   PersistentKeepalive = 15
   ```
   (Lower value = more frequent keepalives)

2. **Check server's MTU settings:**
   ```bash
   # On server, add MTU to wg0.conf
   sudo nano /etc/wireguard/wg0.conf
   ```
   Add under `[Interface]`:
   ```ini
   MTU = 1420
   ```

3. **Restart WireGuard:**
   ```bash
   sudo systemctl restart wg-quick@wg0
   ```

---

### Issue: DNS Not Working Through VPN

**Symptom:** VPN connected, but can't resolve domain names

**Solutions:**
1. **Check client config has `DNS` set:**
   ```ini
   [Interface]
   DNS = 1.1.1.1
   ```

2. **Test DNS manually:**
   ```bash
   nslookup google.com 1.1.1.1
   ```

3. **Restart network services** on client (OS-dependent)

---

### Issue: Firewall Blocking WireGuard

**Symptom:** Can't connect to VPN, but SSH works

**Solutions:**
1. **Verify UFW allows WireGuard port:**
   ```bash
   sudo ufw status | grep 51820
   ```

2. **If missing, add the rule:**
   ```bash
   sudo ufw allow 51820/udp
   ```

3. **Check for cloud provider firewall:**
   - Some VPS providers have **additional firewalls** in their dashboard
   - Log in and ensure UDP port 51820 is allowed

---

### Issue: "Permission Denied" Errors

**Symptom:** Commands fail with permission errors

**Solutions:**
1. **Ensure you're using `sudo`:**
   ```bash
   sudo [command]
   ```

2. **Check file permissions:**
   ```bash
   ls -la /etc/wireguard/
   ```
   Config files should be `600` or `640`

3. **Re-run with correct permissions:**
   ```bash
   sudo chmod 600 /etc/wireguard/wg0.conf
   ```

---

## Security Best Practices

### 1. Regular Updates

**Enable automatic security updates:**

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

**Manually update weekly:**

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2. Install Fail2Ban (SSH Brute-Force Protection)

```bash
# Install Fail2Ban
sudo apt install fail2ban -y

# Create custom SSH jail
sudo nano /etc/fail2ban/jail.local
```

**Add this configuration:**

```ini
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

**Restart Fail2Ban:**

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

---

### 3. Monitor Server Activity

**Install basic monitoring tools:**

```bash
sudo apt install htop nethogs -y
```

**Check active connections:**

```bash
# View WireGuard connections
sudo wg show

# Monitor bandwidth per process
sudo nethogs

# View system resources
htop
```

---

### 4. Backup Your Configuration

**Backup critical files:**

```bash
# Create backup directory
mkdir ~/wireguard-backup

# Backup WireGuard config
sudo cp /etc/wireguard/wg0.conf ~/wireguard-backup/
sudo cp /etc/wireguard/server_privatekey ~/wireguard-backup/
sudo cp /etc/wireguard/server_publickey ~/wireguard-backup/

# Backup SSH config
sudo cp /etc/ssh/sshd_config ~/wireguard-backup/

# Download backups to your local machine
scp -P 2222 ghostvpn@[SERVER_IP]:~/wireguard-backup/* ~/Desktop/
```

⚠️ **Store backups securely** - they contain private keys!

---

### 5. Key Rotation (Advanced)

**Rotate keys every 6-12 months for maximum security:**

1. Generate new server keys (Part 1, Step 4)
2. Update `wg0.conf` with new `PrivateKey`
3. Update all client configs with new server `PublicKey`
4. Restart WireGuard: `sudo systemctl restart wg-quick@wg0`

---

### 6. Server Shutdown When Not in Use

**For maximum privacy**, shut down the server when not using the VPN:

**Via cloud provider dashboard:**
- Log in to your VPS provider
- Select your server → **Power Off**
- Restart when needed (note: IP may change on some providers)

**Cost savings:** Many providers only charge when the server is running.

---

### 7. Two-Factor Authentication (Advanced)

**Add 2FA to SSH for extra security:**

```bash
# Install Google Authenticator
sudo apt install libpam-google-authenticator -y

# Run setup
google-authenticator
```

Follow the prompts, scan the QR code with your authenticator app, and update `/etc/pam.d/sshd`.

---

## Adding More Clients

To connect additional devices (phone, tablet, etc.):

1. **Generate new client keys** (each device should have unique keys)
2. **Create a client config** with a new IP (e.g., `10.0.0.3`, `10.0.0.4`, etc.)
3. **Add the peer on the server:**
   ```bash
   sudo wg set wg0 peer [NEW_CLIENT_PUBLIC_KEY] allowed-ips 10.0.0.3/32
   ```
4. **Save the updated config:**
   ```bash
   sudo wg showconf wg0 | sudo tee /etc/wireguard/wg0.conf
   ```

---

## Useful Commands Reference

```bash
# View WireGuard status
sudo wg show

# View detailed interface info
sudo wg show wg0

# Restart WireGuard
sudo systemctl restart wg-quick@wg0

# View WireGuard logs
sudo journalctl -u wg-quick@wg0 -f

# Check firewall status
sudo ufw status verbose

# View active connections
sudo ss -tulnp

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward

# View NAT rules
sudo iptables -t nat -L -v -n

# Test DNS resolution
nslookup google.com 1.1.1.1

# Check public IP from VPN
curl ifconfig.me
```

---

## Conclusion

Congratulations! 🎉 You now have a **fully functional, secure WireGuard VPN** running on your own server.

**What you've accomplished:**
- ✅ Deployed a WireGuard VPN server
- ✅ Configured client connections
- ✅ Hardened SSH security
- ✅ Set up firewall protection
- ✅ Enabled automatic startup and persistence
- ✅ Implemented best security practices

**Next Steps:**
- Add more devices using the instructions above
- Set up monitoring and logging
- Consider rotating keys periodically
- Explore advanced features like split-tunneling

---

## Resources

- **Official WireGuard Documentation**: [wireguard.com](https://www.wireguard.com/)
- **WireGuard Quick Start**: [wireguard.com/quickstart](https://www.wireguard.com/quickstart/)
- **Ubuntu Server Guide**: [ubuntu.com/server/docs](https://ubuntu.com/server/docs)
- **DigitalOcean WireGuard Tutorial**: [digitalocean.com/community/tutorials](https://www.digitalocean.com/community/tutorials)

---

## License

This guide is released under the MIT License. Feel free to use, modify, and share!

---

## Contributing

Found an error or have a suggestion? Feel free to open an issue or submit a pull request on GitHub!

---

**Stay safe, stay private!** 🔒
