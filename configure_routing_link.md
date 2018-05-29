## Determine Requirements
Identify your ASN and advertised address range. Select an IP address to use as Router ID for BGP processes. Select the interface to use for the routing link. Identify the address ranges to be advertised to neighboring routers.

We'll use the following values in this example:
- Interface:  `eth0`
- ASN:        `65000`
- Router ID:  `172.23.0.127`
- Network(s): `172.23.0.0/24`

## Prepare Interfaces
To begin, we need to prepare the interface that we'll be using to peer with neighboring routers. In the past, this would have meant configuring IPv4 addresses for each side of the connection; however, we will be taking advantage of _unnumbered BGP_ to identify and connect to our peers based on auto-configured IPv6 addresses.

Our main task is to enable auto-configuration for IPv6 and to disable auto-configuration privacy extensions so that our MAC address will be encoded into the _link-local_ IPv6 address for the interface. We also need to ensure that the routing interface will listen for and accept _ICMPv6 router advertisements_ from its peer. Likewise, we need to disable the `dhcpcd` client on the selected interface.

### Configure interface 
Create or edit `/etc/network/interfaces.d/eth0` and ensure that the following IPv6 settings are in place. You may leave IPv4 settings as is or set the interface to manual as shown here.

```
allow-hotplug eth0

iface eth0 inet manual 
iface eth0 inet6 auto
  privext 0
  accept_ra 2
```

### Disable dhcpcd client
Edit `/etc/dhcpcd.conf` and add/edit `denyinterfaces` statement to include `eth0`

## Configure FRRouting
###  Set up address for BGP router ID 
The router can use any locally defined address for the router ID, though it is common to attach the address to loopback interface on the network device to ensure that it is availalbe regardless of which interfaces are online.

We'll perform this configuration within the FRR VTY shell:

```
configure terminal
interface lo
ip address 172.23.0.129/32
quit
```

### Additional unnumbered routing configuration
In order for an unnumbered peer to discover our device, and configure its network stack to forward packets, we need to ensure the interface will send traffic. The following commands will turn on ICMPv6 router advertisements and set an interval of one advertisement every 5 seconds (as recommended by FRR documentation).

Enter the following commands in the FRR VTY shell:
```
configure terminal
interface eth0
ipv6 nd ra-interval 5
no ipv6 nd suppress-ra
quit
```

### Configure BGP Routing
Once the preliminary configuration is complete, we can setup the BGP daemon to peer on the specified interface.

Enter the following commands in VTY shell: 
```
configure terminal
router bgp 65000 
bgp router-id 172.23.0.129 
neighbor eth0 interface remote-as external
network 172.23.0.0/24
quit
```
