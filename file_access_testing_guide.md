# File Access Over VPN Testing Guide

This guide will walk you through testing access to shared files and folders over your VPN connection. It assumes you have already set up file sharing on a target computer as described in the `file_sharing_guide.md`.

## 1. Prerequisites Check

Before you begin testing, ensure the following:

*   **VPN Connected (Home Computer):** Your home computer (the client from which you want to access files) must be successfully connected to the VPN. Verify its VPN connection status and that it has a VPN IP address.
*   **Target Computer Status:**
    *   **Powered On:** The computer hosting the shared files (target computer) must be powered on.
    *   **Connected to VPN:** The target computer must also be connected to the same VPN network and have its own VPN IP address.
    *   **File Sharing Enabled:** File sharing must be configured correctly on the target computer as per the `file_sharing_guide.md`, with the appropriate folders shared and permissions set.
*   **Information Ready:**
    *   **VPN IP of Target Computer:** Know the VPN IP address of the target computer (e.g., `10.8.0.5`). You can usually find this in the VPN server's status log or on the target computer itself.
    *   **Share Name:** Know the exact name of the shared folder (e.g., `MySharedDocs`, `documents`, `JohnsMacDocs`).
    *   **Credentials:** Have the username and password ready for an account on the target computer that has been granted permission to access the share.

## 2. Accessing Shares from Different Client Operating Systems

Follow the instructions specific to your home computer's operating system.

### From Windows

**A. Using File Explorer:**

1.  Open **File Explorer**.
2.  In the address bar at the top, type the path to the share using the target computer's VPN IP address and the share name:
    `\\<VPN-IP-of-target-PC>\<share-name>`
    *   Example: `\\10.8.0.5\MySharedDocs`
3.  Press **Enter**.
4.  If prompted, enter the username and password for an account on the target PC that has permissions to access the share. Remember to prefix the username with the target computer's hostname or domain if necessary (e.g., `TARGETPCNAME\username` or `DOMAIN\username`). Check "Remember my credentials" if desired.

**B. Mapping a Network Drive (Optional, for persistent access):**

1.  Open **File Explorer**.
2.  Right-click on "This PC" (or "Computer" in older Windows versions) in the left-hand navigation pane.
3.  Select "Map network drive..."
4.  Choose an available drive letter (e.g., `Z:`).
5.  In the "Folder" field, type the path to the share: `\\<VPN-IP-of-target-PC>\<share-name>`
    *   Example: `\\10.8.0.5\MySharedDocs`
6.  Check "Reconnect at sign-in" if you want Windows to try to reconnect to this share every time you log in.
7.  Check "Connect using different credentials" if the username/password for the share is different from your Windows login.
8.  Click "Finish." You will be prompted for credentials if you checked the box or if Windows doesn't have cached credentials.

**C. Troubleshooting Tips (Windows):**

*   **Credentials:** Double-check you are using the correct username and password for an account that exists *on the target PC* and has share permissions.
*   **Firewall on Target PC:** Ensure the firewall on the target PC is allowing "File and Printer Sharing" (TCP ports 139, 445; UDP 137, 138) from the VPN IP subnet (e.g., `10.8.0.0/24`).
*   **Share Permissions:** Verify the share permissions *and* the NTFS file system permissions on the target PC allow access for the user account you're using.
*   **Network Discovery:** Ensure Network Discovery is enabled on your home PC for the network profile used by the VPN.
*   **Typo in Path:** Check for typos in the VPN IP address or share name.

### From macOS

**A. Using Finder -> Go -> Connect to Server:**

1.  Open **Finder**.
2.  From the menu bar at the top of the screen, click **Go**, then select **Connect to Server...** (or press Command+K).
3.  In the "Server Address" field, type the protocol, VPN IP address of the target computer, and the share name:
    *   For SMB shares (common for Windows, Linux Samba, modern macOS): `smb://<VPN-IP-of-target-PC>/<share-name>`
        *   Example: `smb://10.8.0.5/MySharedDocs`
    *   For AFP shares (older macOS target): `afp://<VPN-IP-of-target-PC>/<share-name>`
        *   Example: `afp://10.8.0.8/JohnsMacDocs`
4.  Click the "+" button to save the address for future use (optional).
5.  Click **Connect**.
6.  You will be prompted to enter the username and password for an account on the target computer that has permissions to access the share. Choose "Registered User" and enter the credentials.

**B. Troubleshooting Tips (macOS):**

*   **Protocol:** Ensure you're using the correct protocol (`smb://` or `afp://`). SMB is generally preferred.
*   **Credentials:** Verify username and password for the target machine.
*   **Firewall on Target PC:** Check the target PC's firewall settings (Windows Firewall, Linux `iptables`/`firewalld`, or macOS Sharing settings).
*   **Permissions:** Confirm share and file system permissions on the target.
*   **SMB/AFP Enabled on Target:** Ensure the respective service (SMB or AFP) is enabled and configured correctly on the target Mac if it's the sharing machine.

### From Linux

**A. Using a File Manager (e.g., Nautilus on GNOME, Dolphin on KDE):**

1.  Open your file manager.
2.  Look for an option like "Connect to Server," "Other Locations," or an address bar.
3.  Enter the path to the share using the SMB protocol:
    `smb://<VPN-IP-of-target-PC>/<share-name>`
    *   Example: `smb://10.8.0.5/MySharedDocs`
4.  Press Enter or click "Connect."
5.  You'll likely be prompted for credentials (username, domain/workgroup, password) for the target share.

**B. Using `mount` for NFS Shares (if the target is a Linux NFS server):**

1.  Ensure `nfs-common` (Debian/Ubuntu) or `nfs-utils` (CentOS/RHEL) is installed on your Linux client:
    ```bash
    sudo apt update && sudo apt install nfs-common # Debian/Ubuntu
    # or
    sudo yum install nfs-utils # CentOS/RHEL
    ```
2.  Create a local directory to mount the share:
    ```bash
    sudo mkdir /mnt/remote_share
    ```
3.  Mount the NFS share:
    `sudo mount <VPN-IP-of-target-PC>:/path/to/nfs/export /mnt/remote_share`
    *   Example: `sudo mount 10.8.0.7:/srv/nfs/shared_folder /mnt/remote_share`
    *   Note: `/path/to/nfs/export` is the path defined in the `/etc/exports` file on the NFS server.

**C. Using `mount` for SMB Shares (requires `cifs-utils`):**

1.  Ensure `cifs-utils` is installed:
    ```bash
    sudo apt update && sudo apt install cifs-utils # Debian/Ubuntu
    # or
    sudo yum install cifs-utils # CentOS/RHEL
    ```
2.  Create a local mount point:
    ```bash
    sudo mkdir /mnt/smb_share
    ```
3.  Mount the SMB share:
    `sudo mount -t cifs //<VPN-IP-of-target-PC>/<share-name> /mnt/smb_share -o username=<target_username>,uid=$(id -u),gid=$(id -g)`
    *   Example: `sudo mount -t cifs //10.8.0.5/MySharedDocs /mnt/smb_share -o username=target_user`
    *   You will be prompted for the password for `target_username`. You can also specify `password=<password>` in the options, but this is less secure (stores password in bash history).

**D. Troubleshooting Tips (Linux):**

*   **Required Packages:** Make sure `cifs-utils` (for SMB) or `nfs-common` (for NFS) are installed on your client.
*   **Permissions:** File system permissions on the mounted directory and permissions on the remote share.
*   **Firewall on Target PC:** Check firewall on the target (e.g., for Samba ports 139/445 or NFS port 2049 and related services).
*   **NFS Export Settings:** If using NFS, verify the `/etc/exports` configuration on the server allows access from your client's VPN IP or the VPN subnet.
*   **Mount Command Syntax:** Double-check the syntax of your `mount` command.

## 3. Testing Operations

Once you have successfully accessed the share, perform the following tests:

*   **Browsing:**
    *   Can you see the list of files and folders within the shared directory?
    *   Can you navigate into subfolders?

*   **Reading:**
    *   Try opening a simple file (e.g., a `.txt` file, a `.jpg` image, a `.pdf` document). Does it open correctly?
    *   Performance will depend on your VPN connection speed and file size.

*   **Downloading/Copying (from share to your computer):**
    *   Select a file on the remote share.
    *   Copy it (Ctrl+C on Windows/Linux, Command+C on macOS).
    *   Paste it into a folder on your local computer (Ctrl+V or Command+V).
    *   Does the file copy successfully? Check its integrity if it's a large file.

*   **Uploading/Writing (if write permissions were granted on the share):**
    *   **Copying to Share:** Try copying a small file from your local computer to the remote share.
    *   **Creating a Folder:** Try creating a new folder on the remote share.
    *   **Renaming/Deleting:** Try renaming a test file or folder you created, or deleting it.
    *   If these operations fail, it's likely a permissions issue on the target share/filesystem, or the share was mounted as read-only.

## 4. Basic Troubleshooting Checklist

If you encounter problems accessing or using the file share over VPN:

1.  **VPN Connectivity:**
    *   Confirm both your client computer and the target computer are still connected to the VPN.
    *   Try pinging the target computer's VPN IP address from your client computer.
    *   Try pinging your client computer's VPN IP address from the target computer (if possible).
    *   Example: `ping 10.8.0.5` (from client to target)

2.  **Share Names and Paths:**
    *   Double-check that the share name is spelled correctly.
    *   Verify the VPN IP address of the target computer.
    *   Ensure you're using the correct path format for your OS (e.g., `\\server\share` vs `smb://server/share`).

3.  **User Credentials:**
    *   Are you using the correct username and password for an account that exists on the *target machine*?
    *   Does this account have permissions to access the specific share and files/folders?
    *   Try re-entering credentials.

4.  **File/Folder Permissions (on Target Machine):**
    *   Beyond share permissions, check the underlying file system permissions (NTFS permissions on Windows, POSIX permissions on Linux/macOS) on the target machine. The user account needs both share *and* file system permissions.

5.  **Firewall Settings (on Target Machine):**
    *   The most common issue. Ensure the firewall on the computer *hosting the files* is configured to allow incoming traffic for the relevant file sharing service (SMB: TCP 139/445, UDP 137/138; NFS: TCP/UDP 2049 and rpcbind) specifically *from your VPN client's IP address or the entire VPN subnet* (e.g., `10.8.0.0/24`).

6.  **Service Status (on Target Machine):**
    *   Ensure the file sharing service (Samba `smbd` on Linux, "Server" service on Windows, "File Sharing" in macOS System Settings) is running on the target machine.

7.  **VPN Server/Client Logs:**
    *   Check OpenVPN logs on both the client and server for any error messages that might indicate connectivity problems, routing issues, or dropped packets.

By systematically going through these testing steps and troubleshooting tips, you should be able to identify and resolve most issues related to accessing file shares over your VPN.
