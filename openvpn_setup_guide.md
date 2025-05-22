# OpenVPN Server Setup Guide

This guide provides step-by-step instructions for setting up an OpenVPN server on a Linux distribution. You will need root or sudo privileges to perform these actions on your server.

## 1. Installation

First, update your server's package list and install OpenVPN and easy-rsa.

**For Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install openvpn easy-rsa -y
```

**For CentOS:**

You'll need to enable the EPEL (Extra Packages for Enterprise Linux) repository first.

```bash
sudo yum install epel-release -y
sudo yum install openvpn easy-rsa -y
```

## 2. easy-rsa Setup

`easy-rsa` is a utility for managing a Public Key Infrastructure (PKI).

Create a directory for your PKI and navigate into it.

```bash
mkdir ~/easy-rsa
cp -r /usr/share/easy-rsa/* ~/easy-rsa/
cd ~/easy-rsa
```

Initialize the PKI. This creates a `vars` file with default settings. You can edit this file to customize your PKI, but the defaults are usually fine for a basic setup.

```bash
./easyrsa init-pki
```

## 3. Certificate Authority (CA) Generation

Now, build your Certificate Authority (CA). This will create your CA certificate and private key. You'll be prompted for a passphrase for the CA key; make sure to use a strong one and remember it. You will also be asked for a Common Name for your CA (e.g., "MyOpenVPNCA").

```bash
./easyrsa build-ca
```

The CA certificate will be located at `pki/ca.crt` and the CA private key at `pki/private/ca.key`.

## 4. Server Certificate and Key Generation

Next, generate a server certificate and private key. Replace `myserver` with your desired server name. You will be prompted for the CA passphrase you set in the previous step.

```bash
./easyrsa gen-req myserver nopass
```
This creates a certificate request (`.req`) and a private key (`.key`) for the server. The `nopass` option means the server's private key will not be encrypted. This is common for servers so they can start automatically without requiring a passphrase.

Now, sign the server request with your CA:

```bash
./easyrsa sign-req server myserver
```

You'll be asked to confirm the details and then prompted for the CA passphrase.

The server certificate will be at `pki/issued/myserver.crt` and the server private key at `pki/private/myserver.key`.

## 5. Diffie-Hellman Parameters Generation

Generate Diffie-Hellman parameters. This can take some time.

```bash
./easyrsa gen-dh
```

The Diffie-Hellman parameters will be saved to `pki/dh.pem`.

## 6. Server Configuration

Create a server configuration file. OpenVPN's configuration directory is typically `/etc/openvpn/`.

Create `/etc/openvpn/server.conf` with the following content. Make sure to adjust file paths if your `easy-rsa` directory or server name is different.

```ini
port 1194
proto udp
dev tun

# CA, Server Certificate, Server Key, and Diffie-Hellman parameters
ca /home/your_user/easy-rsa/pki/ca.crt # Adjust your_user
cert /home/your_user/easy-rsa/pki/issued/myserver.crt # Adjust your_user and myserver
key /home/your_user/easy-rsa/pki/private/myserver.key # Adjust your_user and myserver
dh /home/your_user/easy-rsa/pki/dh.pem # Adjust your_user

# VPN Subnet
# This defines the virtual IP address pool for clients.
# Choose a subnet that doesn't conflict with your server's LAN or client LANs.
server 10.8.0.0 255.255.255.0

# Redirect all client traffic through the VPN
push "redirect-gateway def1 bypass-dhcp"

# Push DNS servers to clients
# Replace with your preferred DNS servers (e.g., Google DNS, OpenDNS, or your own)
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"

# Keepalive
# Pings remote client every 10 seconds, assumes client is down if no response in 120 seconds.
keepalive 10 120

# Cryptographic Cipher
# AES-256-GCM is a strong and modern cipher.
cipher AES-256-GCM
auth SHA256

# User and Group
# Drop privileges after initialization for better security.
user nobody
group nogroup # Or 'nobody' on some systems

# Status Log
# Logs connection status, useful for monitoring.
status /var/log/openvpn/openvpn-status.log

# Other common options:
# persist-key
# persist-tun
# verb 3 # Verbosity level (0-9)
# explicit-exit-notify 1 # Notify client if server restarts
```

**Explanation of Key Parameters:**

*   `port 1194`: The port OpenVPN listens on (1194 is the standard).
*   `proto udp`: The protocol to use (UDP is generally preferred for VPNs). `tcp` is also an option.
*   `dev tun`: Use a TUN device (creates a routed IP tunnel). `tap` is another option for an Ethernet tunnel.
*   `ca ...`: Path to the CA certificate.
*   `cert ...`: Path to the server's public certificate.
*   `key ...`: Path to the server's private key.
*   `dh ...`: Path to the Diffie-Hellman parameters file.
*   `server 10.8.0.0 255.255.255.0`: Defines the virtual IP address pool for VPN clients. The server will take `10.8.0.1`.
*   `push "redirect-gateway def1 bypass-dhcp"`: Tells clients to route all their internet traffic through the VPN.
*   `push "dhcp-option DNS 8.8.8.8"`: Pushes DNS server addresses to clients.
*   `keepalive 10 120`: Sends a ping every 10 seconds over the control channel; if no response for 120 seconds, the peer is considered down.
*   `cipher AES-256-GCM`: Specifies the encryption cipher. `AES-256-GCM` is a strong and efficient choice.
*   `auth SHA256`: Specifies the HMAC authentication algorithm for control channel packets.
*   `user nobody` / `group nogroup`: Drops OpenVPN's privileges after startup for security.
*   `status /var/log/openvpn/openvpn-status.log`: Path to a file where OpenVPN will log current connections.

**Important:**
*   Copy the necessary PKI files to `/etc/openvpn/` or ensure the paths in `server.conf` point to the correct locations within your `~/easy-rsa/pki/` directory. For production, it's better to copy them to `/etc/openvpn/` and secure permissions. For example:
    ```bash
    sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/
    sudo cp ~/easy-rsa/pki/issued/myserver.crt /etc/openvpn/
    sudo cp ~/easy-rsa/pki/private/myserver.key /etc/openvpn/
    sudo cp ~/easy-rsa/pki/dh.pem /etc/openvpn/
    # Then update paths in server.conf accordingly, e.g.:
    # ca /etc/openvpn/ca.crt
    # cert /etc/openvpn/myserver.crt
    # key /etc/openvpn/myserver.key
    # dh /etc/openvpn/dh.pem
    ```
*   Ensure the user specified by the `user` and `group` directives can access the status log file. You might need to create the `/var/log/openvpn/` directory and set appropriate permissions.
    ```bash
    sudo mkdir -p /var/log/openvpn
    sudo chown nobody:nogroup /var/log/openvpn
    ```

## 7. Firewall Configuration

You need to allow OpenVPN traffic through your server's firewall and enable IP forwarding to allow VPN clients to access the internet through the server.

**Enable IP Forwarding:**

Edit `/etc/sysctl.conf` and uncomment or add the following line:

```
net.ipv4.ip_forward=1
```

Apply the change without rebooting:

```bash
sudo sysctl -p
```

**Firewall Rules (using `ufw` for Ubuntu/Debian):**

```bash
# Allow OpenVPN traffic (default port 1194/udp)
sudo ufw allow 1194/udp

# Allow SSH (if not already allowed)
sudo ufw allow OpenSSH

# Set default policies (optional, but good practice)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# NAT configuration for VPN clients
# Find your primary network interface (e.g., eth0, ens3) using `ip addr` or `ifconfig`
# Edit /etc/ufw/before.rules and add the following lines at the top,
# *before* the *filter rules (lines starting with :ufw-).
# Replace eth0 with your actual public network interface.
#
# BEGIN OPENVPN UFW FORWARDING
# *nat
# :POSTROUTING ACCEPT [0:0]
# -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
# COMMIT
# END OPENVPN UFW FORWARDING

# After editing /etc/ufw/before.rules, enable ufw:
sudo ufw enable
```
To add the NAT rule in `/etc/ufw/before.rules`, you would manually edit the file and add:
```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/24 -o <YOUR_PUBLIC_INTERFACE> -j MASQUERADE
COMMIT
```
Replace `<YOUR_PUBLIC_INTERFACE>` with your server's main network interface (e.g., `eth0`). This section must be added before the `*filter` section in the file.

**Firewall Rules (using `iptables` for CentOS/other systems):**

```bash
# Allow OpenVPN traffic
sudo iptables -A INPUT -p udp --dport 1194 -j ACCEPT

# Allow established and related connections
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# NAT for VPN clients (replace eth0 with your public network interface)
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# Save iptables rules (method varies by distribution)
# For CentOS 7+:
sudo yum install iptables-services -y
sudo systemctl enable iptables
sudo systemctl start iptables
sudo iptables-save | sudo tee /etc/sysconfig/iptables

# For Debian/Ubuntu (if not using ufw):
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

## 8. Starting and Enabling OpenVPN Service

Now you can start the OpenVPN server service. The service name might vary slightly depending on how OpenVPN is packaged for your distribution. It's often `openvpn@server` or `openvpn-server@server`, where `server` refers to the name of your configuration file (e.g., `server.conf`).

**For systems using systemd (most modern Linux distributions):**

```bash
# Start the OpenVPN service for server.conf
sudo systemctl start openvpn@server

# Enable the service to start on boot
sudo systemctl enable openvpn@server

# Check the status of the service
sudo systemctl status openvpn@server

# View logs for troubleshooting
sudo journalctl -u openvpn@server
```

If the service name is `openvpn-server@server.service`:
```bash
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server
sudo systemctl status openvpn-server@server
sudo journalctl -u openvpn-server@server
```

You may need to determine the exact service name. For example, on Ubuntu/Debian, it's typically `openvpn-server@<config_name>.service`. So if your config is `/etc/openvpn/server.conf`, the service is `openvpn-server@server.service`.

```bash
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server
```

This completes the server-side setup. You will then need to generate client certificates and configuration files to allow clients to connect.
