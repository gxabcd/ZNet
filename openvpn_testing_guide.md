# OpenVPN Connectivity Testing Guide

This guide will help you test your OpenVPN setup to ensure clients can connect successfully and that traffic is routing as expected. It assumes your OpenVPN server and client `.ovpn` profiles have been configured according to the previous guides.

## 1. Connecting Clients

Ensure you have transferred the unique `.ovpn` profile to each client device securely.

**Windows:**

1.  **OpenVPN GUI:**
    *   Ensure the OpenVPN GUI application is running.
    *   If you haven't already, place the client's `.ovpn` file into the `config` folder of your OpenVPN installation (e.g., `C:\Program Files\OpenVPN\config` or `C:\Users\[YourUser]\OpenVPN\config`).
    *   Right-click the OpenVPN GUI icon in the system tray.
    *   You should see the profile name (derived from the `.ovpn` filename) in the menu.
    *   Click on the profile and select "Connect."
    *   A log window will appear showing the connection process. If a passphrase was set for the client key, you'll be prompted to enter it.

2.  **OpenVPN Connect:**
    *   Launch the OpenVPN Connect application.
    *   Import the `.ovpn` file if you haven't already (usually via File -> Import Profile or by dragging the file into the app).
    *   Select the imported profile and toggle the switch to connect.

**macOS:**

1.  **Tunnelblick:**
    *   Ensure Tunnelblick is running.
    *   If the `.ovpn` file hasn't been imported, double-click it or drag it onto the Tunnelblick icon in the menu bar. Tunnelblick will install the configuration.
    *   Click the Tunnelblick icon in the menu bar.
    *   Select the desired VPN connection from the list and click "Connect."
    *   You may be prompted for your macOS user password to allow Tunnelblick to manage network settings. If the client key is passphrase protected, Tunnelblick will ask for it.

2.  **Viscosity:**
    *   Open Viscosity.
    *   If not already imported, import the `.ovpn` file (usually by dragging it to the Viscosity icon in the menu bar or via its preferences).
    *   Click the Viscosity icon in the menu bar, select the connection, and click "Connect."

**Linux:**

1.  **Command Line (using `openvpn`):**
    *   Open a terminal.
    *   Run the OpenVPN client with the specific configuration file:
        ```bash
        sudo openvpn --config /path/to/your/client.ovpn
        ```
    *   The terminal will display live log output. Keep this terminal open while the VPN is active. Press `Ctrl+C` to disconnect.

2.  **NetworkManager:**
    *   If you imported the `.ovpn` file into NetworkManager:
        *   Click on the NetworkManager applet (usually in the system tray or top bar).
        *   Go to "VPN Connections" or a similar submenu.
        *   Select the VPN connection you configured and click it to connect.
        *   You might be prompted for your user password or the client key passphrase.

## 2. Checking Connection Status (Client-side)

Once connected, verify the client has obtained a VPN IP address.

**Windows:**

1.  **OpenVPN GUI/Connect:** The client software's log window or status screen should indicate a successful connection and often displays the assigned VPN IP address.
2.  **Command Prompt/PowerShell:**
    *   Open Command Prompt (`cmd`) or PowerShell.
    *   Type `ipconfig`.
    *   Look for a new network adapter, often labeled "Tunnel adapter," "TAP-Windows Adapter V9," or similar. It should have an IP address from the VPN subnet (e.g., `10.8.0.x`).

**macOS:**

1.  **Tunnelblick/Viscosity:** The client application icon in the menu bar usually changes appearance (e.g., turns green) and clicking it will show connection status, including the assigned IP.
2.  **Terminal:**
    *   Open Terminal.
    *   Type `ifconfig` or `ip addr`.
    *   Look for a `tun` or `utun` interface (e.g., `tun0`, `utun0`, `utun1`). It should display the IP address assigned by the VPN server.

**Linux:**

1.  **Command Line (if `openvpn` is running in terminal):** The log output will show messages like `TUN/TAP device tun0 opened` and `Initialization Sequence Completed`.
2.  **Terminal (general):**
    *   Open a new terminal.
    *   Type `ip addr show` or `ifconfig`.
    *   Look for a `tun` interface (e.g., `tun0`). It should display the IP address assigned by the VPN server.
3.  **NetworkManager:** The NetworkManager applet will typically show a lock icon or other indicator that the VPN is active. Clicking on connection details will show the assigned IP.

## 3. Checking Connection Status (Server-side)

**A. OpenVPN Server Logs:**

*   **Status Log (if configured in `server.conf`):**
    The `status /var/log/openvpn/openvpn-status.log` directive in `server.conf` creates a log file that shows current client connections, their virtual IPs, bytes transferred, and when they connected.
    ```bash
    sudo cat /var/log/openvpn/openvpn-status.log
    ```
    Look for entries under "CLIENT LIST".

*   **Systemd Journal (if OpenVPN is run as a systemd service):**
    Replace `server` with your configuration file name (e.g., `openvpn@server` or `openvpn-server@server`).
    ```bash
    sudo journalctl -u openvpn@server
    # To follow logs in real-time:
    sudo journalctl -fu openvpn@server
    ```
    Look for messages indicating client connections, like "client_instance_name/client_ip:port Peer Connection Initiated".

*   **Syslog (older systems or different configurations):**
    OpenVPN might log to `/var/log/syslog` or `/var/log/openvpn.log`.
    ```bash
    sudo tail -f /var/log/syslog | grep openvpn
    sudo tail -f /var/log/openvpn.log
    ```

**B. OpenVPN Management Interface (Optional):**

If you configured the OpenVPN management interface in `server.conf` (e.g., `management localhost 7505`), you can connect to it to get detailed status information.

1.  Connect to the management interface (usually from the server itself for security):
    ```bash
    telnet localhost 7505
    ```
2.  Once connected, you can issue commands like:
    *   `status`: Shows current connections, similar to the status log file.
    *   `load-stats`: Shows server load statistics.
    *   `clientkick <common_name>`: Disconnect a specific client.
    *   `help`: Shows available commands.
    *   `exit` or `quit`: To disconnect from the management interface.

    **Note:** Exposing the management interface externally is a security risk unless properly secured.

## 4. Basic Ping Tests

Once clients are connected and you've verified their VPN IP addresses from the server's status log or management interface, perform these ping tests.

*   **VPN Server's Internal VPN IP:** Typically the first IP in your VPN subnet (e.g., `10.8.0.1` if your `server` directive is `10.8.0.0 255.255.255.0`).
*   **Client's VPN IP:** The IP address assigned to the client by the VPN server (e.g., `10.8.0.2`).

**A. Ping VPN Server from Client:**

*   **From a connected VPN client (Windows, macOS, Linux):**
    Open a terminal or command prompt.
    ```bash
    ping 10.8.0.1 # Replace with your VPN server's internal VPN IP
    ```
    A successful ping indicates basic connectivity to the server over the VPN tunnel.

**B. Ping Another VPN Client from a Client:**

*   This test requires `client-to-client` communication to be enabled on the server. If you added `client-to-client` to your `server.conf`, this should work.
*   **From VPN Client A (e.g., IP `10.8.0.2`):**
    Open a terminal or command prompt.
    ```bash
    ping 10.8.0.3 # Replace with VPN Client B's internal VPN IP
    ```
    If successful, clients can see each other directly over the VPN. If it fails, ensure `client-to-client` is in `server.conf` and the OpenVPN service has been restarted. Also, check client-side firewalls.

**C. Ping Device on Server's Local LAN from Client:**

*   This tests if clients can reach devices on the server's local network (the network the OpenVPN server itself is connected to). This requires appropriate `push "route ..."` directives in the server configuration if you want to access specific LAN subnets beyond just the server itself, and that your server is configured to route/NAT this traffic.
*   **From a connected VPN client:**
    ```bash
    ping <server_lan_ip_address> # e.g., ping 192.168.1.1 (server's actual LAN IP)
    ping <another_device_on_server_lan> # e.g., ping 192.168.1.100
    ```
    *   Pinging the server's own LAN IP should generally work if `redirect-gateway` is used or if specific routes are pushed.
    *   Pinging *other* devices on the server's LAN depends on:
        1.  The server's IP forwarding being enabled.
        2.  Firewall rules on the server allowing this forwarding (NAT/MASQUERADE usually handles this for internet traffic, but internal routing might need specific rules or for the LAN devices to know how to route back to the VPN subnet).
        3.  The LAN devices themselves not having firewalls blocking pings from the VPN subnet.

**D. Ping Client's VPN IP from the Server:**

*   **From the OpenVPN server itself:**
    Open a terminal on the server.
    ```bash
    ping 10.8.0.2 # Replace with the client's assigned VPN IP
    ```
    This should work and confirms the server can reach the client through the tunnel via its virtual IP.

**Troubleshooting Ping Issues:**

*   **Firewalls:** Check firewalls on the client, server, and any intermediate devices. Client-side firewalls (Windows Firewall, macOS Firewall, `iptables` on Linux clients) are common culprits.
*   **Server Configuration:**
    *   Ensure `client-to-client` is present for client-to-client pings.
    *   Ensure IP forwarding is enabled on the server (`net.ipv4.ip_forward=1`).
    *   Ensure firewall rules on the server (iptables/ufw) allow the traffic (e.g., FORWARD chain rules, MASQUERADE for NAT).
*   **Routing:** If pinging LAN devices fails, ensure the server is pushing routes to the clients for those LAN subnets, or that `redirect-gateway` is correctly forcing all traffic through the VPN.
*   **OpenVPN Logs:** Check both client and server OpenVPN logs for any error messages.
*   **Interface Status:** Double-check `ifconfig`/`ip addr` on both client and server to ensure the `tun` interfaces are up and have the correct IP addresses.

This testing guide should help you confirm your OpenVPN setup is working correctly. If you encounter issues, systematically check each point, starting with client connectivity, then server logs, and finally ping tests, while keeping firewalls in mind.
