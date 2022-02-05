
Raspberry pi as Wi-Fi repeater
===============================
[![](https://img.shields.io/github/license/ArieLevs/Raspberry-WiFi-Repeater.svg)](https://github.com/ArieLevs/Raspberry-WiFi-Repeater/blob/master/LICENSE)

This is a simple guide to set Raspberry pi as Wi-Fi repeater, 
I know there are many guides out there, [including the official](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/configuration/access-point-routed.adoc), 
but none of them worked 100% without modifications.

Based on: `Linux raspberrypi 5.10.63-v7+` (Raspbian 10 buster)

In order to achieve our goal we will need to set up two separate interfaces,
You will need additional Wi-Fi usb device or use the ethernet connection.

Let's assume your home lan address is 10.0.0.0/24,
We will extend this network using another lan at address 10.0.1.0/24.

This guide assumes the OS is a **clean** installation and running, with the Wi-Fi\ethernet interface to the router "10.0.0.0/24" is set up.
```text
                                              wlan0 interface is being configured
                                                    (10.0.22.0/24 network)
# Option 1                                                    |
                                                              |
                 +- Router ----+          +--- Raspberry ---+ /        +- Laptop ----+
                 | DHCP server |          | 10.0.0.18       |/         | WLAN Client |
(Internet)---WAN-+             +---LAN----|/       wlan0 AP +-)))  (((-+             |
                 | 10.0.0.0/24 |          |        10.0.22.1|\         | 10.0.22.36  |
                 +-------------+          +-----------------+ |        +-------------+
                                                              |                 
                                                         new network
                                                         DHCP server
# Option 2                                                    |
                 +- Router ----+          +--- Raspberry ---+ |        +- Laptop ----+
                 | DHCP server |   WiFi   | 10.0.0.18       |/         | WLAN Client |
(Internet)---WAN-+             +-)))  (((-|/       wlan0 AP +-)))  (((-+             |
                 | 10.0.0.0/24 |          |        10.0.22.1|          | 10.0.22.36  |
                 +-------------+          +-----------------+          +-------------+

--------------------------------------------------------------------------------------

# Option 3 (extended network)
                 +- Router ----+          +--- Raspberry ---+          +- Laptop ----+
                 | DHCP server |          | 10.0.0.18 (eth0)|          | WLAN Client |
(Internet)---WAN-+             +---LAN----|/       wlan0 AP +-)))  (((-+             |
                 |             |          |                 |\         |             |
                 | 10.0.0.0/24 |          | br eth0<->wlan0 | \        | 10.0.0.36/24|
                 +-------------+          +-----------------+  \       +-------------+
                                                               |
                                                               |
                                                wlan0 interface is being configured
                                               (extend original 10.0.0.0/24 network)
```

Automatic Installation via Ansible:
-----------------------------------
Execute from project root directory  
(note for the `,` below, or the ip/dns will be considered a hosts inventory filename)
the configured device ip from above example is `10.0.0.18`

* if updating country variable make sure country is a [valid Alpha-2 code ISO 3166-1 country code](https://en.wikipedia.org/wiki/ISO_3166-1)
* Custom access point CIDR can be provided by passing these variables: (view [defaults](ansible/roles/setup_raspberry_access_point/defaults/main.yaml))
  ```
  ap_ip_addr="10.0.1.1/24"
  ap_start_ip_addr="10.0.1.2"
  ap_end_ip_addr="10.0.1.100"
  ap_subnet_mask="255.255.255.0"
  ```
### options **1** OR **3** install example:
With option one/three you bridge `eth0 <-> wlan0` and probably connected to raspberry pi via LAN Ethernet cable (to ip `10.0.0.18`),
so only the configured access point name/pass should be provided 
(if not provided default `ap_ssid_name` and `ap_ssid_pass` will be set to `nalkinscloud`)
```shell
ansible-playbook -u pi --ask-pass \
    -i "10.0.0.18," \
    ansible/setup_repeater.yaml \
    -e ap_ssid_name=AP_SSID \
    -e ap_ssid_pass=AP_PASSWORD \
    -e country="FI"
    ## add below variable to deploy as an extended network
    #-e extended_network=True
```

### option **2** install example:
With option two/four you bridge `wlan0 <-> wlan1` and probably want to ssh to raspberry via the Wi-Fi interface (`10.0.0.8`),
in this case you should manually set up `wpa_supplicant` as described below.  
If you connected via ethernet cable (to lets say `10.0.0.37`) update relevant ip host.  
the `ssid_name` and `ssid_pass` should contain your home Wi-Fi access.
```shell
ansible-playbook -u pi --ask-pass \
    -i "10.0.0.18," \
    ansible/setup_repeater.yaml \
    -e ap_ssid_name="AP_SSID>" \
    -e ap_ssid_pass="<AP_PASSWORD>" \
    -e ssid_name="<LAN_SSID>" \
    -e ssid_pass="<LAN_PASSWORD>"
```
* default password for `pi` user is: `raspberry`

Manual Installation:
--------------------
When doing `ip a` you should have 4\3 interfaces,
loopback, eth0 for lan cable connection, wlan0 is the onboard Wi-Fi
and wlan1 is the usb Wi-Fi interface if connected.

wlan1 or eth0 will connect to your home LAN at 10.0.0.0/24  
wlan0 will create a new network using 10.0.1.0/24, or it will extend your current 10.0.0.0/24 network if you choose to. 
then a bridge (br0) will bridge the interfaces.

for option 2 **only** set up (not needed when using eth0 connection to router), 
provide your home SSID and password `sudo vi /etc/wpa_supplicant/wpa_supplicant.conf`
```text
# update with relevant country
country=US
network={
    ssid="YOUR_HOME_SSID"
    psk="WIFI_PASSWORD"
}
```

Once raspberry was able to connect to the router (with internet access), do
```shell
sudo apt-get update
sudo apt-get upgrade
```

Configure Access Point (hostapd)
--------------------------------
Install hostapd
```shell
sudo apt-get install hostapd
```
type `sudo vi /etc/hostapd/hostapd.conf`
And append
```shell
# update with relevant country
country_code=US
interface=wlan0

## un-comment below line if setting up extended network (options 3)
# bridge=br0

## comment out below line if setting extended network (options 3)
driver=nl80211

ssid=[AP_SSID]
hw_mode=g
channel=6
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=[AP_PASS]
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Execute `sudo vi /etc/default/hostapd` and replace line #DAEMON_CONF with
```text
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Ensure wireless operation ([comply regulations](https://wireless.wiki.kernel.org/en/developers/regulatory/statement))
```shell
sudo rfkill unblock wlan
```

Start the service
```shell
sudo systemctl unmask hostapd
sudo systemctl start hostapd
```

Configure DHCP (dhcpcd)
--------------------------------
Configure `sudo vi /etc/dhcpcd.conf`
* if setting extended network (option 3)
```shell
## set dhcpcd configure only br0 via DHCP
## Add to start of file

denyinterfaces wlan0 eth0

## THIS IS CURRENTLY NOT SUPPORTED
## with additional wifi, add below 3 lines
#denyinterfaces wlan0 wlan1
#interface wlan0
#  nohook wpa_supplicant

## Add to end of file
interface br0
```
* if setting routed network (option 1/2)
```shell
## configure a static IP for wlan1 (or eth0) and static IP for wlan0
## Add to the end of file lines

## If using static IP with additional wifi (optional)
#interface wlan1
#    static ip_address=10.0.0.2/24
#    static routers=10.0.0.1
#    static domain_name_servers=10.0.0.1 8.8.8.8

## If using static IP with ethernet (optional)
#interface eth0
#    static ip_address=10.0.0.2/24
#    static routers=10.0.0.1
#    static domain_name_servers=10.0.0.1 8.8.8.8

interface wlan0
    static ip_address=10.0.1.1/24
    nohook wpa_supplicant
```
Restart the service `sudo service dhcpcd restart`

Add Extended Network 
--------
* If setting an extended network (options 3/4):

Install bridge-utils
```shell
sudo apt-get install bridge-utils
```

Execute `sudo vi /etc/network/interfaces.d/raspberry_interfaces`
and add
```shell
auto br0
iface br0 inet dhcp
bridge_ports eth0 wlan0

## THIS IS CURRENTLY NOT SUPPORTED
## or if setting wlan1 with wlan0 (option 4)
#bridge_ports wlan1 wlan0
```

All set! clients not should be able to connect to the Wi-Fi network chosen at `[AP_SSID]` and get an IP address from extended router.

Add Routing - Repeater - only for routed network (options 1/2)
-----------

Install dnsmasq
```shell
sudo apt-get install dnsmasq
```

#### Set up the DHCP server (dnsmasq)
```shell
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo vi /etc/dnsmasq.conf
```
And append to above file
```shell
interface=wlan0
    dhcp-range=10.0.1.2,10.0.1.100,255.255.255.0,12h
```

start the service
```shell
sudo systemctl start dnsmasq
```

#### Add routing
```shell
sudo apt-get install iptables
```

type `sudo vi /etc/sysctl.conf` and uncommon line:
```text
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```
Then run
```shell
## if using eth0 as connection to router
#inter=eth0
inter=wlan1

sudo iptables -t nat -A POSTROUTING -o ${inter} -j MASQUERADE
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Edit `sudo vi /etc/rc.local` and append above "exit 0" line 
```shell
iptables-restore < /etc/iptables.ipv4.nat
```

* **only** if setting `wlan1` append to `/etc/network/interfaces.d/raspberry_interfaces` file:
    ```
    # Bridge setup
    auto br0
    iface br0 inet manual
    bridge_ports wlan1 wlan0
    ```
Then `sudo reboot`