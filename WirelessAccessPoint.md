## Setting up a Raspberry Pi as an access point in a standalone network (NAT)
In order to work as an access point, the Raspberry Pi will need to have access point software installed, along with DHCP server software to provide connecting devices with a network address.

To create an access point, we'll need DNSMasq and HostAPD. Install all the required software in one go with this command:
```
sudo apt install dnsmasq hostapd
```
Since the configuration files are not ready yet, turn the new software off as follows:
```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

### Configuring a static IP

We are configuring a standalone network to act as a server, so the Raspberry Pi needs to have a static IP address assigned to the wireless port. This documentation assumes that we are using the standard 192.168.x.x IP addresses for our wireless network, so we will assign the server the IP address 192.168.4.1. It is also assumed that the wireless device being used is `wlan0`.

To configure the static IP address, edit the dhcpcd configuration file with:
```
sudo nano /etc/dhcpcd.conf
```
Go to the end of the file and edit it so that it looks like the following:
```
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```

Now restart the dhcpcd daemon and set up the new `wlan0` configuration:
```
sudo service dhcpcd restart
```

### Configuring the DHCP server (dnsmasq)

The DHCP service is provided by dnsmasq. By default, the configuration file contains a lot of information that is not needed, and it is easier to start from scratch. Rename this configuration file, and edit a new one:
```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```
Type or copy the following information into the dnsmasq configuration file and save it:
```
interface=wlan0      # Use the require wireless interface - usually wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```
So for `wlan0`, we are going to provide IP addresses between 192.168.4.2 and 192.168.4.20, with a lease time of 24 hours. If you are providing DHCP services for other network devices (e.g. eth0), you could add more sections with the appropriate interface header, with the range of addresses you intend to provide to that interface.

There are many more options for dnsmasq; see the [dnsmasq documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html) for more details.

Reload `dnsmasq` to use the updated configuration:
```
sudo systemctl reload dnsmasq
```

### Configuring the access point host software (hostapd)
You need to edit the hostapd configuration file, located at /etc/hostapd/hostapd.conf, to add the various parameters for your wireless network. After initial install, this will be a new/empty file.
```
sudo nano /etc/hostapd/hostapd.conf
```
Add the information below to the configuration file. This configuration assumes we are using channel 7, with a network name of NameOfNetwork, and a password AardvarkBadgerHedgehog. Note that the name and password should **not** have quotes around them. The passphrase should be between 8 and 64 characters in length.

To use the 5 GHz band, you can change the operations mode from hw_mode=g to hw_mode=a. Possible values for hw_mode are:

- a = IEEE 802.11a (5 GHz)
- b = IEEE 802.11b (2.4 GHz)
- g = IEEE 802.11g (2.4 GHz)
- ad = IEEE 802.11ad (60 GHz)
```
interface=wlan0
driver=nl80211
ssid=NameOfNetwork
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=AardvarkBadgerHedgehog
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
We now need to tell the system where to find this configuration file.
```
sudo nano /etc/default/hostapd
```
Find the line with #DAEMON_CONF, and replace it with this:
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
### Start it up
Now enable and start `hostapd`:
```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```
Do a quick check of their status to ensure they are active and running:
```
sudo systemctl status hostapd
sudo systemctl status dnsmasq
```
### Add routing and masquerade
Edit /etc/sysctl.conf and uncomment this line:
```
net.ipv4.ip_forward=1
```
Add a masquerade for outbound traffic on eth0:
```
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
```
Save the iptables rule.
```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
Edit /etc/rc.local and add this just above "exit 0" to install these rules on boot.
```
iptables-restore < /etc/iptables.ipv4.nat
```
Reboot and ensure it still functions.

Using a wireless device, search for networks. The network SSID you specified in the hostapd configuration should now be present, and it should be accessible with the specified password.

If SSH is enabled on the Raspberry Pi access point, it should be possible to connect to it from another Linux box (or a system with SSH connectivity present) as follows, assuming the `pi` account is present:
```
ssh pi@192.168.4.1
```
By this point, the Raspberry Pi is acting as an access point, and other devices can associate with it. Associated devices can access the Raspberry Pi access point via its IP address for operations such as `rsync`, `scp`, or `ssh`.
