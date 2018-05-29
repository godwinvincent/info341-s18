# Set up Raspberry Pi as a Wireless Access Point

## Prerequisites
Install `isc-dhcp-server` and create a subnet definition that will serve your wireless LAN. If you plan to replace a current, ethernet-based subnet, it's recommended that you complete the initial configuration with a new subnet (so as not to lock yourself out of the management interface).

## Install the hostapd service
- `sudo apt update`
- `sudo apt install hostapd`

## Halt the hostapd service
- `sudo systemctl stop hostapd`

## Select network parameters
- _SSID:_ The broadcasted name of your network
- _PASSPHRASE:_ The passphrase used to connect to your network


## Set a static IP address for wlan0
By default, Raspbian is using `dhcpcd` to manage network interfaces; however, this utility does not work smoothly with the `isc-dhcp-server`. Edit `/etc/dhcpcd.conf` and add/edit `denyinterfaces` statement to include `wlan0`.

Create a new, static configuration for the interface in `/etc/network/interfaces.d/wlan0`.
```
allow-hotplug wlan0
iface wlan0 inet static
  address 172.23.0.1/25
```

## Create a the PSK for your network
Create a hashed PSK from your chosen passphrase by running `wpa_passphrase "_SSID_"`.

At the prompt, enter the PASSPHRASE you selected earlier.

Copy the hex encoded PSK and insert it into a new file called `/etc/hostapd-psk` as shown here:
```
00:00:00:00:00:00 HEX_PSK
```
### Restrict permissions on the PSK
`sudo chmod 640 /etc/hostapd-psk`


## Configure hostapd 
We'll create our access point configuration at `/etc/hostapd/hostapd.conf`:
```
interface=wlan0
driver=nl80211
ssid=SSID
ieee80211n=1
wmm_enabled=1
hw_mode=g
channel=6
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_psk_file=/etc/hostapd-psk
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
rsn_pairwise=CCMP
```
You can test this configuration by disabling the Wi-Fi client on wlan0 with `sudo systemctl stop wpa_supplicant` and manually running `sudo hostapd /etc/hostapd/hostapd.conf`. Check the output for errors and confirm that your access point is now visible on the network.

## Set hostapd configuration file location
Edit `/etc/default/hostapd` and set `DAEMON_CONF="/etc/hostapd/hostapd.conf"` (the line is commented out by default).`

## Confirm network interface configuration
Review the network configuration that you defined earlier. Ensure that parameters are valid and that it does not conflict with other interfaces. Reboot the device now to ensure that the new configuration is applied.

## Confirm DHCP configuration
Review the configuration for your DHCP service and confirm that you have created a valid subnet definition that corresponds to the static configuration of wlan0. Confirm that the dhcp daemon is listening on wlan0 (in `isc-dhcp-server` this is done in /etc/default/isc-dhcp-server`. Check for errors in your modified configuration by running `sudo systemctl restart isc-dhcp-server` to ensure that the service can restart smoothly.

## Disable the wpa_supplicant
While sometimes possible to operate a wireless interface in Access Point (AP) mode and managed/client mode simultaneously, we're going to avoid that can of worms for now. Disable the `wpa_supplicant` by running `sudo systemctl disable wpa_supplicant`. 

## Confirm NAT configuration 
Assuming your WLAN exists within a private subnet, you will rely on NAT to access the public Internet. Verify that your current MASQUERADE rules will apply to traffic originating from `wlan0` and exiting on a public-facing interface (e.g., `eth0`).

## Confirm firewall configuration
Your firewall must be configured to allow packets to be forwarded from the new WLAN other attached networks. As a starting point, confirm that you will accept traffic originating from `wlan0` as well as any return traffic that is associated with those sessions. 

## Enable the hostapd service
When the system is ready, you can instruct systemd to launch `hostapd` at boot with `sudo systemctl enable --now hostapd`. By adding the -now switch, we will the also start the service immediately without the need for a reboot. 
