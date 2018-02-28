# RASPBERRY PI 3 - WIFI STATION+AP+Tor Router

Running the Raspberry Pi 3 as a Wifi client (station) and access point (ap) from the single built-in wifi, In addition to enable Tor for the access point clients.

Its been written about before, but this way is better.  The access point device is created before networking
starts (using udev) and there is no need to run anything from `/etc/rc.local`.  No reboot, no scripts.

## Use Cases

The Rpi 3 wifi chipset can support running an access point and a station function simultaneously.  One
use case is a device that connects to the cloud (the station, via a user's home wifi network) but
that needs an admin interface (the access point) to configure the network.  The user powers on the
device, then logs into the access point using a specified SSID/password.  The user runs a browser
and connects to the access point IP address (or hostname), which is running a web server to configure
the station network (the user's wifi).

Another use case might be to create a guest interface to your home wifi.  You can configure the client
side with your wifi particulars, then configure the access point with a password you can give out to your
guests.  When the party's over, change the access point password.

# Setting up interfaces

## /etc/network/interfaces.d/ap

`sudo nano /etc/network/interfaces.d/ap`

Then add these lines:

    allow-hotplug uap0
    auto uap0
    iface uap0 inet static
        address 10.3.141.1
        netmask 255.255.255.0

## /etc/network/interfaces.d/station

`sudo nano /etc/network/interfaces.d/station`

Then add these lines:

    allow-hotplug wlan0
    iface wlan0 inet manual
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf


## /etc/udev/rules.d/90-wireless.rules

`sudo nano /etc/udev/rules.d/90-wireless.rules`

Then add these lines:

    ACTION=="add", SUBSYSTEM=="ieee80211", KERNEL=="phy0", \
        RUN+="/sbin/iw phy %k interface add uap0 type __ap"

## Do not let DHCPCD manage wpa_supplicant!!

`sudo rm -f /lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant`

## Set up the client wifi (station) on wlan0.

Create `/etc/wpa_supplicant/wpa_supplicant.conf`.

`sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`

The contents depend on whether your home network is open, WEP or WPA.  It is probably WPA, and so should look like:

    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    country=GB
    
    network={
	    ssid="_ST_SSID_"
	    scan_ssid=1
	    psk="_ST_PASSWORD_"
	    key_mgmt=WPA-PSK
    }

Replace `_ST_SSID_` with your home network SSID and `_ST_PASSWORD_` with your wifi password (in clear text).

## Restart DHCPCD

`sudo systemctl restart dhcpcd`
	
## Bring up the station (client) interface

`sudo ifup wlan0`
	
At this point your client wifi should be connected.

## Manually invoke the udev rule for the AP interface.

Execute the command below.  This will also bring up the `uap0` interface.  It will wiggle the network, so you might be kicked off (esp. if you
are logged into your Pi on wifi).  Just log back on.

`sudo /sbin/iw phy phy0 interface add uap0 type __ap`

# Setting up DNS, Access Point and Firewall rules

## Install these packages

    sudo apt-get update
    sudo apt-get install hostapd dnsmasq iptables-persistent

## /etc/dnsmasq.conf

`sudo nano /etc/dnsmasq.conf`

Then add these lines:

    interface=lo,uap0
    no-dhcp-interface=lo,wlan0
    bind-interfaces
    server=8.8.8.8
    dhcp-range=10.3.141.50,10.3.141.255,12h

## /etc/hostapd/hostapd.conf

`sudo nano /etc/hostapd/hostapd.conf`

Then add these lines:

    interface=uap0
    ssid=_AP_SSID_
    hw_mode=g
    channel=6
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=_AP_PASSWORD_
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP

Replace `_AP_SSID_` with the SSID you want for your access point.  Replace `_AP_PASSWORD_` with the password for your access point.  Make sure it has
enough characters to be a legal password!  (8 characters minimum).

## /etc/default/hostapd

`sudo nano /etc/default/hostapd`

Change `DAEMON_CONF=" "` to this line:

    DAEMON_CONF="/etc/hostapd/hostapd.conf"

# Finalizing

## Now restart the dns and hostapd services

    sudo systemctl restart dnsmasq
    sudo systemctl restart hostapd

## Restart the client interface

I dunno.  The client interface went down for some reason (see below "bringup order").  Bring it back up:

    sudo ifdown wlan0
    sudo ifup wlan0

## Permanently deal with interface bringup order

    # see https://unix.stackexchange.com/questions/396059/unable-to-establish-connection-with-mlme-connect-failed-ret-1-operation-not-p
	
Edit `/etc/rc.local` and add the following lines just before "exit 0":

    sleep 30
    ifdown wlan0
    sleep 10
    rm -f /var/run/wpa_supplicant/wlan0
    ifup wlan0

## Bridge AP to cient side

This is optional.  If you do this step, then someone connected to the AP side can browse the internet through the client side.

    sudo echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
    sudo echo 1 > /proc/sys/net/ipv4/ip_forward
    sudo iptables -t nat -A POSTROUTING -s 10.3.141.0/24 ! -d 10.3.141.0/24 -j MASQUERADE
    sudo iptables-save > /etc/iptables/rules.v4

# Setting up TOR

## Install Tor

`sudo apt-get install tor -y`

## Configure Tor

Open /etc/tor/torrc
 
`sudo nano /etc/tor/torrc`

Add the following configurations at the end of this file, This will configure TOR to run on port 9050 and port 53.

	Log notice file /var/log/tor/notices.log
	VirtualAddrNetwork 10.192.0.0/10
	AutomapHostsSuffixes .onion,.exit
	AutomapHostsOnResolve 1
	TransPort 9040
	TransListenAddress 10.3.141.1
	DNSPort 53
	DNSListenAddress 10.3.141.1

## Using Bridges

If you want to use Tor brigdes install these packages

	sudo apt-get install obfs4proxy obfsproxy

Add the following lines

	UseBridges 1
	ClientTransportPlugin obfs3 exec /usr/bin/obfsproxy managed
	ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy managed

Get TOR bridges from here https://bridges.torproject.org/options and add it to /etc/tor/torrc like this

	Bridge obfs3 194.132.209.182:35115 69CBC731FD77DC2200E3C1FC333018A0D078C446
	Bridge obfs3 194.132.209.105:33647 108E3F93A2E9A772D4EABF4DA88688D2066885AF
	Bridge obfs3 54.242.5.211:40872 769FD4B52E3286132CE00F1983F8B69EF5996A22

## Setting up iptables

With TOR now set up, we need to flush the iptables, run these commands

	sudo iptables -F
	sudo iptables -t nat -F

Applying new IP Tables:
This will route all the traffic incoming from the uap0 connection through TOR connection using port 53.
The first line will add an exception for port 22 since we need that to be able to SSH to the Raspberry Pi.

	sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 22 -j REDIRECT --to-ports 22
	sudo iptables -t nat -A PREROUTING -i uap0 -p tcp --dport 22 -j REDIRECT --to-ports 22
	sudo iptables -t nat -A PREROUTING -i uap0 -p udp --dport 53 -j REDIRECT --to-ports 53
	sudo iptables -t nat -A PREROUTING -i uap0 -p tcp --syn -j REDIRECT --to-ports 9040

If you need to check that the IP tables have been correctly entered you can use this command.

`sudo iptables -t nat -L`

## Saving new iptables

With our new iptables rules in place we need to store this into the file we set up in our wireless access point, this will ensure the new IP Tables are loaded instead.

`sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"`

## Enable TOR logging

Now lets create our log file, this will be handy for tracking problems.

	sudo touch /var/log/tor/notices.log
	sudo chown debian-tor /var/log/tor/notices.log
	sudo chmod 644 /var/log/tor/notices.log

## Starting TOR Router

Now we can finally fire up the TOR service.

`sudo service tor start`

You can check that the service is running by using the following command

`sudo service tor status`

And to check if TOR works well run these commands and compare your ip

`curl ifconfig.me`

`torify curl ifconfig.me 2>/dev/null`

Finally, let’s make the TOR service start on boot, this will ensure that the traffic will always be routed through it.

`sudo update-rc.d tor enable`

That's it, you should be good to go. You should not have needed to reboot your Pi, but if you do then everything you did will remain in place and functional.
