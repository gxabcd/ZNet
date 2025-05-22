# File Sharing Over VPN Guide

This guide explains how to set up and access shared files and folders on computers connected via an OpenVPN (or any other) VPN.

## 1. General Principles

*   **Same Logical Network:** Once connected to the VPN, your computer and the remote computers (also connected to the same VPN) are effectively on the same logical network. This means they can "see" each other using their VPN IP addresses.
*   **Standard Methods Apply:** You'll use the standard file sharing methods native to each operating system (OS). The main difference is that you'll use the VPN IP address of the target machine instead of its regular local network IP.
*   **User Accounts & Permissions:** For security, always use user accounts and configure permissions for shared resources. Avoid "everyone" or "guest" access where possible, especially over a network. Only grant the necessary level of access (e.g., read-only vs. read/write).
*   **VPN IP Addresses:** You will need to know the VPN IP address of the machine hosting the files. This is the IP address assigned by the VPN server (e.g., from the `10.8.0.0/24` range in our OpenVPN server example).

## 2. Windows (SMB/CIFS)

Windows uses the SMB/CIFS protocol for file sharing.

**A. How to Share a Folder:**

1.  **Locate Folder:** Open File Explorer, right-click the folder you want to share, and select "Properties."
2.  **Sharing Tab:** Go to the "Sharing" tab.
3.  **Advanced Sharing:** Click "Advanced Sharing..."
4.  **Share this folder:** Check the "Share this folder" box.
5.  **Share name:** Give the share a name (e.g., `MySharedDocs`). This is the name clients will use to connect.
6.  **Permissions:** Click "Permissions."
    *   By default, "Everyone" might have Read access. It's best to remove "Everyone."
    *   Click "Add..." and type the usernames of Windows accounts on the sharing PC that should have access (e.g., `John`, `vpn_users_group`). Click "Check Names" and "OK."
    *   For each user or group, select them and then check "Allow" for "Full Control," "Change," or "Read" permissions as needed.
    *   Click "OK" on the Permissions window and "OK" on the Advanced Sharing window.
7.  **Network Discovery & File Sharing:** Ensure Network Discovery and File and Printer Sharing are enabled in your firewall settings for the "Private" network profile (which your VPN connection should ideally use).
    *   Go to Control Panel -> Network and Sharing Center -> Change advanced sharing settings.
    *   Expand the "Private" profile (or the profile your VPN uses).
    *   Turn on "Network discovery" and "Turn on file and printer sharing."

**B. Accessing the Share from Another Windows Machine (VPN Client):**

1.  Open File Explorer.
2.  In the address bar, type `\\<VPN-IP-of-target-PC>\<share-name>`
    *   Example: `\\10.8.0.5\MySharedDocs` (assuming `10.8.0.5` is the VPN IP of the PC sharing the folder).
3.  Press Enter. You'll be prompted for the username and password of an account on the target PC that has been granted share permissions.

**C. Firewall Considerations (on the PC sharing the files):**

*   The Windows Firewall must allow incoming connections for "File and Printer Sharing." This usually involves allowing traffic on TCP ports 139 and 445, and UDP ports 137 and 138.
*   When enabling "File and Printer Sharing" through the Windows Defender Firewall GUI, it typically creates the necessary rules. Ensure these rules apply to the network profile your VPN connection is using (often "Private" or sometimes "Public" if not configured correctly).
*   It's good practice to scope these firewall rules to only allow connections from your VPN IP subnet (e.g., `10.8.0.0/24`).

## 3. Linux (Samba for SMB/CIFS, or NFS)

### Samba (for compatibility with Windows/macOS)

Samba allows Linux to share files using the SMB/CIFS protocol.

**A. Install Samba:**

*   **Debian/Ubuntu:**
    ```bash
    sudo apt update
    sudo apt install samba -y
    ```
*   **CentOS/RHEL/Fedora:**
    ```bash
    sudo yum install samba samba-client -y # For older systems
    # or
    sudo dnf install samba samba-client -y # For newer systems
    ```

**B. Basic `smb.conf` Configuration:**

Edit the Samba configuration file: `/etc/samba/smb.conf`.
Make a backup first: `sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak`

Add a share definition at the end of the file. For example:

```ini
[documents]
    comment = Shared Documents via VPN
    path = /srv/samba/documents  # Create this directory: sudo mkdir -p /srv/samba/documents
    browseable = yes
    guest ok = no                # Disallow guest access
    read only = no               # Allow write access (set to 'yes' for read-only)
    valid users = @sambausers    # Only users in the 'sambausers' group (or specific users like 'user1', 'user2')
    # Or, for a specific user: valid users = yourlinuxuser
    create mask = 0664
    directory mask = 0775
    # Ensure the owner of /srv/samba/documents is appropriate, e.g.:
    # sudo chown -R yourlinuxuser:sambausers /srv/samba/documents
    # sudo chmod -R g+w /srv/samba/documents
```

*   Adjust `path` to the directory you want to share.
*   `valid users`: You can list specific users or use a group (e.g., `@sambausers`). If using a group, ensure users are added to this system group.

**C. Create Samba Users:**

Samba uses its own password database. Users must exist as system users first.
Replace `yourlinuxuser` with an actual system username.

```bash
sudo smbpasswd -a yourlinuxuser
```
You'll be prompted to set a Samba password for this user. This password will be used by clients to connect to the share.

**D. Restart Samba Service:**

```bash
sudo systemctl restart smbd
sudo systemctl enable smbd # To start on boot
# On older systems:
# sudo service smbd restart
# sudo chkconfig smbd on
```

**E. Accessing Samba Share:**

*   **From Windows:** Same as accessing a Windows share: `\\<Linux-VPN-IP>\<sharename>` (e.g., `\\10.8.0.6\documents`).
*   **From macOS:** Finder -> Go -> Connect to Server, then `smb://<Linux-VPN-IP>/documents`.
*   **From Linux:**
    *   Using file manager (e.g., Nautilus, Dolphin): Type `smb://<Linux-VPN-IP>/documents` in the address bar.
    *   Using command line (requires `cifs-utils` or `samba-client` package):
        ```bash
        sudo apt install cifs-utils # Debian/Ubuntu
        sudo yum install cifs-utils # CentOS/RHEL
        sudo mkdir /mnt/samba_share
        sudo mount -t cifs //<Linux-VPN-IP>/documents /mnt/samba_share -o username=yourlinuxuser,uid=$(id -u),gid=$(id -g)
        # You'll be prompted for the Samba password for 'yourlinuxuser'.
        ```

### NFS (Network File System - common for Linux-to-Linux)

NFS is a traditional way to share files between Unix-like systems.

**A. Install NFS Server:**

*   **Debian/Ubuntu:**
    ```bash
    sudo apt update
    sudo apt install nfs-kernel-server -y
    ```
*   **CentOS/RHEL/Fedora:**
    ```bash
    sudo yum install nfs-utils -y # Or dnf
    ```

**B. Configure Exports:**

Edit the `/etc/exports` file to define which directories are shared and who can access them.

```bash
sudo mkdir -p /srv/nfs/shared_folder # Create the directory to share
# Example: sudo chown nobody:nogroup /srv/nfs/shared_folder (for basic security)
```

Add a line to `/etc/exports`:
`/srv/nfs/shared_folder <VPN-client-IP-or-subnet>(rw,sync,no_subtree_check)`

*   Example for a specific client IP: `/srv/nfs/shared_folder 10.8.0.10(rw,sync,no_subtree_check)`
*   Example for the entire VPN subnet: `/srv/nfs/shared_folder 10.8.0.0/24(rw,sync,no_subtree_check)`

Options:
*   `rw`: Read/Write access. Use `ro` for Read-Only.
*   `sync`: Reply to requests only after changes have been committed to stable storage.
*   `no_subtree_check`: Disables subtree checking, which can have mild security implications but is often needed for reliability.

Apply changes:
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server # Debian/Ubuntu
sudo systemctl enable nfs-kernel-server

sudo systemctl restart nfs-server # CentOS/RHEL
sudo systemctl enable nfs-server
```

**C. Accessing NFS Share from a Linux Client:**

1.  Install NFS client package:
    ```bash
    sudo apt install nfs-common -y # Debian/Ubuntu
    sudo yum install nfs-utils -y  # CentOS/RHEL
    ```
2.  Create a mount point:
    ```bash
    sudo mkdir /mnt/nfs_share
    ```
3.  Mount the share:
    ```bash
    sudo mount <NFS-Server-VPN-IP>:/srv/nfs/shared_folder /mnt/nfs_share
    # Example: sudo mount 10.8.0.7:/srv/nfs/shared_folder /mnt/nfs_share
    ```
4.  To mount automatically on boot, add an entry to `/etc/fstab` on the client:
    `<NFS-Server-VPN-IP>:/srv/nfs/shared_folder /mnt/nfs_share nfs defaults 0 0`

## 4. macOS (SMB or AFP)

macOS supports both SMB (recommended for cross-platform compatibility) and AFP (Apple Filing Protocol, mainly for older Macs).

**A. Enable File Sharing:**

1.  Open **System Settings** (on newer macOS) or **System Preferences** (on older macOS).
2.  Go to **General -> Sharing** (newer macOS) or click on "Sharing" (older macOS).
3.  Check the box for "File Sharing."
4.  Click the "i" icon next to File Sharing (newer macOS) or the "Options..." button (older macOS) to ensure "Share files and folders using SMB" is checked. AFP can also be enabled if needed.

**B. Add Shared Folders and Configure Permissions:**

1.  Under "File Sharing," click the "+" button under the "Shared Folders" list to add a folder you want to share.
2.  Select the folder in the list. In the "Users" list on the right:
    *   Add users or groups from your macOS system.
    *   Set permissions for each user/group (Read & Write, Read Only, No Access).

**C. Accessing macOS Share:**

*   **From another macOS:**
    1.  Open Finder.
    2.  Go to Menu Bar -> Go -> Connect to Server (or press Command+K).
    3.  Enter `smb://<macOS-VPN-IP>` or `afp://<macOS-VPN-IP>`.
        *   Example: `smb://10.8.0.8`
    4.  Click "Connect." You'll be prompted for a username and password for an account on the sharing Mac.

*   **From Windows:**
    1.  File Explorer address bar: `\\<macOS-VPN-IP>\<share-name>`
        *   You might need to know the exact share name as defined on the Mac if it's different from the folder name. Often, it's the user's home folder or folders explicitly shared.
        *   Example: `\\10.8.0.8\JohnsMacDocs`

*   **From Linux (Samba client):**
    *   File manager: `smb://<macOS-VPN-IP>`
    *   Command line (mount):
        ```bash
        sudo mount -t cifs //<macOS-VPN-IP>/ShareName /mnt/mac_share -o username=macuser,uid=$(id -u),gid=$(id -g)
        ```

## 5. Firewall Considerations (General)

On any machine that is sharing files, its local firewall must be configured to allow incoming connections for the specific file sharing protocol from the VPN client IP addresses (or the entire VPN subnet).

*   **Windows (SMB):**
    *   TCP ports 139, 445.
    *   UDP ports 137, 138.
    *   Usually managed by enabling "File and Printer Sharing" in Windows Defender Firewall.

*   **Linux (Samba - SMB):**
    *   TCP ports 139, 445.
    *   UDP ports 137, 138.
    *   Use `ufw` (Ubuntu/Debian) or `firewalld`/`iptables` (CentOS/RHEL/Fedora).
        *   `sudo ufw allow Samba` or `sudo ufw allow from 10.8.0.0/24 to any app Samba`
        *   `sudo firewall-cmd --permanent --add-service=samba` (then `sudo firewall-cmd --reload`)

*   **Linux (NFS):**
    *   TCP and UDP port 2049 (for NFS itself).
    *   Additional ports for `rpcbind`, `mountd`, `statd` if they are not handled automatically or if you have a very restrictive firewall. Often, allowing port 2049 and ensuring `rpcbind` is accessible from the VPN subnet is sufficient.
        *   `sudo ufw allow from 10.8.0.0/24 to any port 2049`
        *   `sudo firewall-cmd --permanent --add-service=nfs` (then `sudo firewall-cmd --reload`)

*   **macOS:**
    *   The built-in macOS firewall ("Stealth Mode" or "Block all incoming connections" should be off, or specific exceptions added for file sharing).
    *   When you enable "File Sharing" in System Settings/Preferences, macOS usually adjusts the firewall automatically for built-in services. If using third-party firewall software, configure it accordingly.

**Best Practice:** When configuring firewall rules, be as specific as possible. Instead of allowing traffic from "any" source, restrict it to your VPN IP address range (e.g., `10.8.0.0/24`). This enhances security by ensuring that only devices connected to your VPN can attempt to access the file shares.

Remember to restart relevant services (Samba, NFS) after making configuration changes and to test access from a client machine connected to the VPN.
