
<img src="https://i.imgur.com/6XRP3gB.png" width="100" height="100" />

# WireGuard Server for Windows
WS4W is a desktop application that allows running and managing a WireGuard server endpoint on Windows.

Inspired by Henry Chang's post, [How to Setup Wireguard VPN Server On Windows](https://www.henrychang.ca/how-to-setup-wireguard-vpn-server-on-windows/), my goal was to create an application that automated and simplified many of the complex steps. While still not quite a plug-and-play solution, the idea is to be able to perform each of the prerequisite steps, one-by-one, without running any scripts, modifying the Registry, or entering the Control Panel.

# Getting Started
The latest release is available [here](https://github.com/micahmo/WireGuardServerForWindows/releases/latest). Download the installer and run.

> **Note**: The application will request to run as Administrator. Due to all the finagling of the registry, Windows services, wg.exe calls, etc., it is easier to run the whole application elevated.

#### Upgrade from 1.5.2
Before introducing an installer, WS4W was distributed as a portable application. The portable versions (1.5.2 and earlier) have no automatic upgrade path to the installer version. To upgrade, simply delete the downloaded portable version and download the installer. No configuration settings will be lost.

# What Does It Do?

Below are the tasks that can be performed automatically using this application.

## Before
![BeforeScreenshot](https://i.imgur.com/Mlyd0TS.png)

### WireGuard.exe
This step downloads and runs the latest version of WireGuard for Windows from https://download.wireguard.com/windows-client/wireguard-installer.exe. Once installed, it can be uninstalled directly from WS4W, too.

### Server Configuration
![ServerConfiguration](https://user-images.githubusercontent.com/7417301/137597967-5dfcf8ba-5a22-4dcf-98f9-3aed21ae3c5e.png)

Here you can configure the server endpoint. See the WireGuard documentation for the meaning of each of these fields. The Private Key, Public Key, and Preshared Key are generated by calling `wg genkey`, `wg pubkey [private key]`, and `wg genpsk`, respectively.

> **Note**: It is important that the server's network range not conflict with the host system's IP address or LAN network range.

In addition to creating/udpating the configuration file for the server endpoint, editing the server configuration will also update the `ScopeAddress` registry value (under `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters`). This is the IP address that is used for the WireGuard adapter when using the Internet Sharing feature (explained [here](#internet-sharing)). Thus, the Address property of the server configuration serves to determine the allowable addresses for clients, as well as the IP that Windows will assign to the WireGuard adapter when performing Internet Sharing. Note the IP address is grabbed from the `ScopeAddress` at the time when Internet Sharing is first performed. That means that if the server's IP address is changed in the configuration (and thus the `ScopeAddress` registry value is updated), the WireGuard interface will no longer accurately reflect the desired server IP. Therefore, WS4W will prompt to re-share internet. If canceled, Internet Sharing will be disabled and will have to be re-enabled manually.

> **Important**: You must configure port forwarding on your router. Forward all UDP traffic that is destined for your server endpoint port (default `51820`) to the LAN IP of your server. Every router is different, so it is difficult to give specific guidance here. As an example, here is what the port forwarding rule would look like on a Verizon Quantum Gateway router.
> 
> ![](https://user-images.githubusercontent.com/7417301/127727564-0d666c41-4998-4c5d-8d2a-e7b730e545c8.png)

You should set the Endpoint property to your public IPv4, IPv6, or domain address, followed by whatever port you have forwarded. The `Detect Public IP Address` button will attempt to detect your public address automatically using the [ipify.org](https://ipify.org) API. However, if possible, it is recommended that you use a domain name with DDNS. That way, if your public IP address changes, your clients will be able to find your server endpoint without reconfiguration.

### Client Configuration
![ClientConfiguration](https://i.imgur.com/frxdJ7S.png)

Here you can configure the client(s). The Address can be entered manually or calculated based on the server's network range. For example, if the server's network is `10.253.0.0/24`, the client config can determine that `10.253.0.2` is a valid address. Note that the first address in the range (in this example, `10.253.0.1`) is reserved for the server. DNS is optional, but recommended. Lastly, the Private Key and Public Keys are again generated using `wg genkey` and `wg pubkey [private key]`. However, the Preshared Key must match the server's. If it has already been generated in the server config, it can be automatically copied to the client config.

Once configured, it's easy to import the configuration into your client app of choice via QR code or by exporting the `.conf` file.

![ClientQrCode](https://i.imgur.com/IOIQ1Rx.png)

### Tunnnel Service
Once the server and client(s) are configured, you may install the tunnel service, which creates a new network interface for WireGuard using the `wireguard /installtunnelservice` command. After installation, the tunnel may be also removed directly within WS4W. This uses the `wireguard /uninstalltunnelservice` command.

After completing this step, WireGuard clients should be able to get as far as performing a successful handshake with the server.

> **Note:** If the server configuration is edited after the tunnel service is installed, the tunnel service will automatically be updated via the `wg syncconf` command (if the newly saved server configuration is valid). This is also true of the client configurations, updates to which often cause the server configuration to be updated (e.g., if a new client is added, the server configuration must be aware of this new peer).

### Private Network
Even after the tunnel service is installed, some protocols may be blocked. It is recommended to change the network profile to `Private`, which eases Windows restrictions on the network.

> **Note**: On a system where the shared internet connection originates from a domain network, this step is not necessary, as the WireGuard interfaces picks up the profile of the shared domain network.

### Internet Sharing
![InternetSharing](https://i.imgur.com/GCKoVIZ.png)

Perhaps most importantly, internet sharing must be enabled in order to provide a real network connection to the WireGuard interface. In Windows, this is accomplished using Internet Connection Sharing, which serves as NAT router between the system's public network and the devices connected to the WireGuard interface.

When configuring this option, you may select any of your network adapters to share. Note that it will likely only work for adapters whose status is `Connected`, and it will only be useful for adapters which provide internet or LAN access. When choosing the adapter to share, hover over the menu item to get more details, including the adapter's assigned IP address, to determine if it's the one you want to share.

> **Note:** When performing internet sharing, the WireGuard adapter is assigned an IP from the `ScopeAddress` registry value (under `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters`). This value is automatically set when updating the Address property of the server configuration. See more [here](#server-configuration).

### Persistent Internet Sharing
There are issues in Windows that cause Internet Sharing to become disabled after a reboot. If the WireGuard server is intended to be left unattended, it is recommended to enable Persistent Internet Sharing so that no interaction is required after rebooting.

When enabling this feature, two actions are performed in Windows:
1. The `Internet Connection Sharing` service startup mode is changed from `Manual` to `Automatic`.
2. The value of the `EnableRebootPersistConnection` regstry value in `HKLM\Software\Microsoft\Windows\CurrentVersion\SharedAccess` is set to `1` (it is created if not found).

Even with these workarounds, Internet Sharing can become disabled after a reboot. Therefore, one more action is performed. A Scheduled Task is created that disables and re-enables Internet Sharing using the WS4W CLI upon system boot. This should be sufficient to guarantee that sharing remains enabled.

### View Server Status
![ServerStatus](https://i.imgur.com/dcSJXKU.png)

Once the tunnel is installed, the status of the WireGuard interface may be viewed. This is accomplished via the `wg show` command. It will be continually updated as long as `Update Live` is checked.

## After
![AfterScreenshot](https://user-images.githubusercontent.com/7417301/152651829-e31d2c1d-666e-426b-be55-dd73a6f3c913.png)

## CLI
There is also a CLI bundled in the portable download called `ws4w.exe` which can be invoked from a terminal or called from a script. In addition to messages written to standard out, the CLI will also set the exit code based on the success of executing the given command. In PowerShell, for example, the exit code can be printed with `echo $lastexitcode`.

> **Note**: The CLI must also be run as an Administrator for the same reasons as above.

### Usage
The CLI uses verbs, or top-level commands, each of which has its own set of options. You can run `ws4w.exe --help` for a list of all verbs or `ws4w.exe verb --help` to see the list of options for a particular verb.

#### List of Supported Verbs
* ```ws4w.exe restartinternetsharing [--network <NETWORK_TO_SHARE>]```
	* This will tell WS4W to attempt to restart the Internet Sharing feature.
	* The `--network` option may be passed to specify which network WS4W should share.
	* If Internet Sharing is already enabled, WS4W will attempt to reshare the same network (unless `--network` is passed).
	* If multiple networks are already shared, it is not possible to tell which one is shared with the WireGuard network, so the `--network` option must be passed to specify.
	* If Internet Sharing is not already enabled, the `--network` option must be passed, otherwise there is no way to know which network to share.
	* The exit code will be 0 if the requested or previously shared network was successfully reshared.
      > This command is used by the Scheduled Task that is created when Persistent Internet Sharing is enabled.
* ```ws4w.exe setpath```
    * This will tell WS4W to add the current executing directory to the system's `PATH` environment variable.
    * This verb has no options.
      > This command is used by the installer when the "Add CLI to PATH" option is selected.

# Known Issues

### Inability to Enable Internet Sharing
If you experience the following error message when enabling Internet Sharing, please perform the following manual steps.

![image](https://user-images.githubusercontent.com/7417301/145692723-f90e6e95-4628-4725-a44d-54377097f883.png)

 - Open Network Connections in the Control Panel.
 - Right-click > Properties on the network interface that you want to share.
    - Go to the Sharing tab and check "Allow other network users to connect through this computer's Internet connection".
	- From the "Home networking connection" dropdown, choose `wg_server`.
	- Press OK.
 - Close and reopen WS4W. It should now show Internet Sharing enabled, and subsequent attempts to disable/re-enable should be sucessful going forward.

> Note: This issue is often triggered after creating a new virtual switch for a VM. The manual workaround should only be needed once after that and does not affect the virtual switch.
