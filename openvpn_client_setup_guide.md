# OpenVPN Client Setup Guide

This guide explains how to generate client configuration files (`.ovpn`) for connecting to an OpenVPN server set up according to our `openvpn_setup_guide.md`. These steps are typically performed on the server where `easy-rsa` and the CA are located.

## 1. Generating Client Certificates and Keys

For each client that needs to connect to the VPN, you must generate a unique certificate and private key.

**On the server where you set up easy-rsa:**

Navigate to your easy-rsa directory (e.g., `~/easy-rsa/`).

```bash
cd ~/easy-rsa
```

To generate a certificate and key for a client (e.g., `client1`):

```bash
# The 'nopass' option creates an unencrypted private key for the client.
# If you omit 'nopass', the client will need to enter a passphrase when connecting.
./easyrsa build-client-full client1 nopass
```

You will be prompted for the CA passphrase that you set when creating the CA.

The client's certificate will be located at `pki/issued/client1.crt` and its private key at `pki/private/client1.key`. The CA certificate is at `pki/ca.crt`.

Repeat this process for each client, using a unique name (e.g., `client2`, `laptop-user`, etc.).

## 2. Creating a Base Client Configuration File (`.ovpn`)

Create a template file, for example, `client.ovpn` or `base.conf`. This file will be customized for each client.

```ini
client
dev tun
proto udp # Should match the server's protocol

# Replace <server-ip> with your OpenVPN server's public IP address or domain name.
# Replace <port> with the port your OpenVPN server is listening on (default 1194).
remote <server-ip> <port>

resolv-retry infinite
nobind

# These options help ensure the client reconnects smoothly if the connection drops.
persist-key
persist-tun

# Specify the CA certificate, client certificate, and client key.
# These will often be embedded directly into the .ovpn file (see next section).
# ca ca.crt
# cert client.crt
# key client.key

# Verify the server certificate is signed by your CA and is a server certificate.
remote-cert-tls server

# Cryptographic Cipher and Authentication
# These MUST match the server's configuration.
cipher AES-256-GCM
auth SHA256

# Optional: Add to improve DNS handling on some systems, especially Windows.
# pull-filter ignore "dhcp-option DNS"
# script-security 2
# up /etc/openvpn/update-resolv-conf
# down /etc/openvpn/update-resolv-conf

# Optional: Verbosity level
# verb 3
```

**Explanation of Key Parameters:**

*   `client`: Declares this is a client configuration.
*   `dev tun`: Use a TUN device (must match server).
*   `proto udp`: Protocol (must match server).
*   `remote <server-ip> <port>`: The public IP address or hostname of your OpenVPN server and the port it's listening on. **This is the most important line to customize.**
*   `resolv-retry infinite`: If DNS resolution fails for the server's hostname, retry indefinitely.
*   `nobind`: Don't bind to a specific local port. Allows multiple clients on the same machine.
*   `persist-key`: Keep the same key across restarts.
*   `persist-tun`: Keep the TUN/TAP device open across restarts.
*   `ca <ca-file>`: Path to the CA certificate file (if not embedded).
*   `cert <cert-file>`: Path to the client's public certificate file (if not embedded).
*   `key <key-file>`: Path to the client's private key file (if not embedded).
*   `remote-cert-tls server`: Verifies that the server's certificate has the "TLS Web Server Authentication" extended key usage. This is a security measure to prevent man-in-the-middle attacks.
*   `cipher AES-256-GCM`: Encryption cipher (must match server).
*   `auth SHA256`: Authentication algorithm (must match server).

## 3. Embedding Certificates and Keys into the `.ovpn` File

For ease of distribution, it's common to embed the CA certificate, client certificate, and client private key directly into the `.ovpn` file. This creates a single, self-contained file for each client.

To do this, modify the client configuration template as follows:

```ini
client
dev tun
proto udp
remote <server-ip> <port> # CHANGE THIS
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256
# verb 3 # Optional verbosity

<ca>
-----BEGIN CERTIFICATE-----
(Contents of your CA certificate: pki/ca.crt)
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
(Contents of the client's certificate: pki/issued/client_name.crt)
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
(Contents of the client's private key: pki/private/client_name.key)
-----END PRIVATE KEY-----
</key>

# Optional: TLS Crypt Auth (ta.key)
# If you generated a ta.key on the server and configured it with `tls-auth ta.key 0`
# and on the client with `tls-auth ta.key 1`, you need to include it here.
# key-direction 1
# <tls-auth>
# -----BEGIN OpenVPN Static key V1-----
# (Contents of ta.key)
# -----END OpenVPN Static key V1-----
# </tls-auth>
```

**Steps to Create an Embedded `.ovpn` File for "client1":**

1.  **Create a copy** of your base client configuration file (e.g., `client1.ovpn`).
2.  **Replace `<server-ip>` and `<port>`** with your server's actual IP/hostname and port.
3.  **Get the CA certificate content:**
    ```bash
    cat ~/easy-rsa/pki/ca.crt
    ```
    Copy the output (including `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----`) and paste it between the `<ca>` and `</ca>` tags in `client1.ovpn`.
4.  **Get the client certificate content:**
    ```bash
    cat ~/easy-rsa/pki/issued/client1.crt
    ```
    Copy the output and paste it between the `<cert>` and `</cert>` tags.
5.  **Get the client private key content:**
    ```bash
    cat ~/easy-rsa/pki/private/client1.key
    ```
    Copy the output and paste it between the `<key>` and `</key>` tags.

You will now have a `client1.ovpn` file that contains all necessary information.

## 4. Client Software Installation

Clients will need OpenVPN client software installed on their devices.

*   **Windows:**
    *   Official OpenVPN Community Client: [OpenVPN Downloads Page](https://openvpn.net/community-downloads/)
    *   OpenVPN GUI (a popular third-party GUI): [OpenVPN GUI GitHub](https://github.com/OpenVPN/openvpn-gui/releases)
*   **macOS:**
    *   Tunnelblick: [Tunnelblick Website](https://tunnelblick.net/) (free, open-source)
    *   Viscosity: [Viscosity Website](https://www.sparklabs.com/viscosity/) (commercial)
*   **Linux:**
    *   Usually available via package managers:
        ```bash
        # Debian/Ubuntu
        sudo apt install openvpn

        # Fedora/CentOS/RHEL
        sudo yum install openvpn
        # or
        sudo dnf install openvpn
        ```
    *   NetworkManager often has OpenVPN integration: `network-manager-openvpn` and `network-manager-openvpn-gnome` (or KDE equivalent).
*   **iOS/Android:**
    *   Search for "OpenVPN Connect" in the Apple App Store or Google Play Store.

## 5. Importing/Using the `.ovpn` File

The method for importing the `.ovpn` file varies by client software:

*   **OpenVPN GUI (Windows):** Place the `.ovpn` file in the `config` directory of your OpenVPN installation (usually `C:\Program Files\OpenVPN\config` or `C:\Users\[YourUser]\OpenVPN\config`). Then, right-click the OpenVPN GUI icon in the system tray and choose "Connect."
*   **Tunnelblick (macOS):** Double-click the `.ovpn` file, or drag it onto the Tunnelblick icon in the menu bar. Tunnelblick will then install the configuration.
*   **Viscosity (macOS/Windows):** Drag and drop the `.ovpn` file into the Viscosity window or import it through its preferences/menu.
*   **Linux (command line):**
    ```bash
    sudo openvpn --config /path/to/client1.ovpn
    ```
*   **Linux (NetworkManager):** Import the `.ovpn` file through the NetworkManager GUI (Network Settings -> VPN -> Add VPN -> Import from file...).
*   **iOS/Android (OpenVPN Connect):** Transfer the `.ovpn` file to your device (e.g., via email, USB, cloud storage) and open it with the OpenVPN Connect app. The app will prompt you to import the profile.

## 6. Secure Distribution

The `.ovpn` file contains sensitive information, including the client's private key and the CA certificate. It is crucial to transfer this file to the client in a secure manner.

*   **Avoid sending `.ovpn` files via unencrypted email.**
*   Use secure methods like:
    *   **SCP or SFTP:** Securely copy the file directly to the client machine if you have SSH access.
    *   **Encrypted USB drive:** Physically hand over the file.
    *   **Password-protected archive:** Compress the `.ovpn` file into an encrypted ZIP or 7z archive and share the password through a separate secure channel (e.g., phone call, encrypted message).
    *   **Secure file sharing services:** Use services that offer end-to-end encryption.

Each client must receive their *own unique* `.ovpn` file. Do not reuse the same client certificate and key for multiple clients if you want to be able to manage them individually (e.g., revoke access for one client without affecting others).

This concludes the client setup guide. Your clients should now be able to connect to the OpenVPN server.
