# OpenVPN and File Sharing Security Hardening Guide

This guide provides a checklist of security best practices to harden your OpenVPN setup and protect your file shares. Following these recommendations will significantly reduce the risk of unauthorized access and data breaches.

## 1. Strong Credentials

*   **VPN Client Certificate Passphrases:**
    *   If client private keys (`.key` files) are encrypted (i.e., you did not use `nopass` during `build-client-full`), use strong, unique passphrases for each.
    *   A strong passphrase is long (15+ characters), includes a mix of uppercase letters, lowercase letters, numbers, and symbols.
*   **User Account Passwords:**
    *   Enforce strong, unique passwords for all user accounts that can access file shares.
    *   These passwords should be different from VPN certificate passphrases or any other accounts.
*   **Avoid Defaults:**
    *   Never use default usernames or passwords for any service (OS, file sharing software, etc.).
    *   Change any default credentials immediately during setup.
*   **Password Manager:**
    *   Consider using a reputable password manager to generate and store complex credentials securely.

## 2. Software Updates

*   **OpenVPN Server Software:**
    *   Regularly check for and apply updates to the OpenVPN server software package. Subscribe to security mailing lists for your distribution or OpenVPN project to be notified of vulnerabilities.
    *   Example (Ubuntu/Debian): `sudo apt update && sudo apt upgrade openvpn`
*   **OpenVPN Client Software:**
    *   Ensure all client machines are running the latest version of their OpenVPN client software (e.g., OpenVPN GUI, Tunnelblick, OpenVPN Connect).
*   **Operating Systems:**
    *   Keep the OS of the OpenVPN server and all client machines (Windows, macOS, Linux) updated with the latest security patches. Enable automatic updates where appropriate.
*   **File Sharing Software:**
    *   Regularly update file sharing services (e.g., Samba for SMB/CIFS, NFS utilities).
    *   Example (Samba on Ubuntu/Debian): `sudo apt update && sudo apt upgrade samba`

## 3. Firewall Configuration

*   **VPN Server Firewall (e.g., `ufw`, `firewalld`, `iptables`):**
    *   **Principle of Least Access:** Only allow incoming connections on the specific port your OpenVPN server is listening on (e.g., UDP port 1194).
    *   Allow SSH (TCP port 22) *only if necessary* for management, and ideally restrict SSH access to known IP addresses or use key-based authentication.
    *   Explicitly drop or reject all other unsolicited incoming traffic.
    *   **NAT/Forwarding Rules:** Review IP forwarding and NAT rules (e.g., `iptables` MASQUERADE rule) to ensure they are not overly permissive. They should only allow traffic from your VPN subnet to the intended destinations (e.g., the internet or specific internal networks).
*   **File Sharing Host Firewalls (on computers sharing files):**
    *   Only allow incoming connections for the specific file sharing protocols (e.g., SMB TCP ports 139/445, NFS TCP/UDP port 2049).
    *   **Crucially, restrict the source of these connections to your VPN IP address range** (e.g., `10.8.0.0/24`).
    *   **Never expose file sharing ports directly to the public internet.** The VPN is your secure entry point.
*   **Client Firewalls:**
    *   Ensure local firewalls (Windows Defender Firewall, macOS Firewall, `ufw`/`firewalld` on Linux clients) are active and configured on all client machines.

## 4. OpenVPN Server Hardening

*   **Strong Cryptographic Settings:**
    *   Use strong ciphers and authentication digests as recommended in the `openvpn_setup_guide.md`.
        *   `cipher AES-256-GCM` (or `AES-128-GCM`)
        *   `auth SHA256` (or stronger like SHA384/SHA512)
    *   Ensure your Diffie-Hellman parameters (`dh.pem`) are sufficiently strong (e.g., 2048-bit or higher).
*   **Disable Unused Features:**
    *   If certain OpenVPN features are not needed, disable them in `server.conf` to reduce the attack surface.
*   **Run as Non-Root User:**
    *   Use `user nobody` and `group nogroup` (or a dedicated unprivileged user) in `server.conf`. This limits the potential damage if the OpenVPN process is compromised.
*   **Management Interface:**
    *   If the management interface is enabled, restrict access to localhost (`management localhost 7505`) and protect it with a strong password file if external access is absolutely required (which is generally discouraged).
*   **`tls-auth` or `tls-crypt`:**
    *   Implement `tls-auth ta.key 0` (server) and `tls-auth ta.key 1` (client) or `tls-crypt ta.key` to add an extra layer of HMAC authentication to control channel packets. This helps protect against DoS attacks, port scanning, and buffer overflows in the TLS stack.
    *   Generate `ta.key` using `openvpn --genkey --secret ta.key`.
*   **Limit Concurrent Connections:**
    *   Use `max-clients <number>` in `server.conf` if you want to limit the number of concurrent VPN connections.

## 5. Certificate Management

*   **Protect CA Key:**
    *   The Certificate Authority private key (`ca.key` in your `easy-rsa/pki/private/` directory) is the most critical component. If compromised, all issued certificates are untrusted.
    *   **Keep `ca.key` offline if possible.** Store it on an encrypted USB drive in a secure location. Only bring it online when you need to sign new client or server certificates.
    *   The CA machine itself should be highly secured.
*   **One Certificate Per Client:**
    *   Use a unique certificate for each client device/user. This allows for granular control and revocation. Do not share `.ovpn` files (which contain client certificates and keys) between users.
*   **Certificate Revocation:**
    *   If a client certificate is compromised, lost, or a user should no longer have access, revoke their certificate immediately.
    *   **Revocation Process (using easy-rsa):**
        1.  Navigate to your `easy-rsa` directory.
        2.  Run `./easyrsa revoke client_name` (e.g., `./easyrsa revoke client1`).
        3.  Generate an updated Certificate Revocation List (CRL): `./easyrsa gen-crl`.
        4.  Copy the generated `crl.pem` (usually found in `pki/crl.pem`) to your OpenVPN server directory (e.g., `/etc/openvpn/`).
        5.  Add or uncomment the `crl-verify /etc/openvpn/crl.pem` directive in your `server.conf`.
        6.  Restart the OpenVPN server.
    *   Periodically update the CRL on the server.

## 6. File Share Permissions (Principle of Least Privilege)

*   **Minimum Necessary Permissions:**
    *   When configuring file shares (SMB/NFS), grant users and groups only the permissions they absolutely need.
    *   If a user only needs to read files, grant read-only access, not read/write or full control.
*   **Avoid "Guest" or "Everyone" Access:**
    *   Be extremely cautious with granting access to "Everyone," "Guest," or anonymous users, especially write access. Whenever possible, require authenticated access.
*   **NTFS/POSIX Permissions:**
    *   Remember that effective permissions are often a combination of share permissions *and* underlying file system permissions (NTFS on Windows, POSIX on Linux/macOS). Ensure both are configured correctly.
*   **Regular Audits:**
    *   Periodically review file share permissions to ensure they are still appropriate.

## 7. Logging and Monitoring

*   **OpenVPN Server Logs:**
    *   Regularly review OpenVPN server logs (e.g., `/var/log/openvpn/openvpn-status.log`, `journalctl -u openvpn@server`, or `/var/log/syslog`) for:
        *   Repeated failed connection attempts.
        *   Connections from unexpected IP addresses (if your clients have static public IPs).
        *   Unusual error messages.
    *   Set `verb 3` or `verb 4` in `server.conf` for a reasonable level of logging. Higher verbosity can be used for troubleshooting but generates more data.
*   **File Sharing Logs:**
    *   Enable and regularly check logs on the file-sharing machines (e.g., Samba logs in `/var/log/samba/`, Windows Event Logs for file access). Look for failed login attempts or unauthorized access patterns.
*   **System Logs:**
    *   Monitor general system logs on the server for any signs of compromise or unusual activity.
*   **Alerts:**
    *   Consider setting up automated alerts for critical security events, such as multiple failed VPN logins, CA compromise alerts (if your monitoring system can detect CA key usage), or high numbers of failed file share access attempts. Tools like `fail2ban` can be configured to automatically block IPs after repeated failed login attempts.

## 8. Physical Security (for On-Premise Servers)

*   If your OpenVPN server or file sharing servers are hosted on-premise:
    *   Ensure the server machines are located in a physically secure environment (e.g., locked room, server rack).
    *   Restrict physical access to authorized personnel only.
    *   Protect against environmental threats (power outages, cooling, etc.).

## 9. Secure Backup of Configurations and Data

*   **OpenVPN Server Configuration:**
    *   Securely back up your OpenVPN `server.conf` file.
    *   **Crucially, back up your entire `easy-rsa` PKI directory, especially `ca.key`, `ca.crt`, server certificates/keys, and the CRL (`crl.pem`). Store these backups encrypted and in a separate, secure location (ideally offline and off-site).**
*   **File Share Data:**
    *   Implement a robust backup strategy for the actual data stored on your file shares.
    *   Test your backup and restore procedures regularly.
*   **Encryption for Backups:**
    *   Ensure all sensitive backups (PKI, configurations, shared data) are encrypted both in transit and at rest.

By diligently applying these security practices, you can create a more resilient and secure OpenVPN and file sharing environment. Security is an ongoing process, so regularly review and update your configurations and practices.
