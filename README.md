# NOTE NOTE NOTE

If you are using a modern version of rasbian (ie stretch), then please visit: https://github.com/peebles/rpi3-wifi-station-ap-stretch

# RASPBERRY PI 3 - WIFI STATION+AP

Running the Raspberry Pi 3 as a Wifi client (station) and access point (ap) from the single built-in wifi.

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

## /etc/network/interfaces

    source-directory /etc/network/interfaces.d
    
    auto lo
    iface lo inet loopback

    allow-hotplug wlan0
    iface wlan0 inet manual
    	wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf

    allow-hotplug uap0
    auto uap0
    iface uap0 inet static
     	address 192.168.10.1
    	netmask 255.255.255.0

    auto eth0
    iface eth0 inet static
    	address 192.168.100.250
    	netmask 255.255.255.0
     	network 192.168.100.0
     	broadcast 192.168.100.255
    	gateway 192.168.100.1
        dns-nameservers 8.8.8.8 8.8.4.4

    iface default inet dhcp


## /etc/udev/rules.d/90-wireless.rules 

    ACTION=="add", SUBSYSTEM=="ieee80211", KERNEL=="phy0", \
        RUN+="/sbin/iw phy %k interface add uap0 type __ap"

## Manually invoke the rule.

Execute the command below.  This will also bring up the `uap0` interface.  It will wiggle the network, so you might be kicked off (esp. if you
are logged into your Pi on wifi).  Just log back on.

    /sbin/iw phy phy0 interface add uap0 type __ap
	
## Set up the client wifi (station) on wlan0.

Create `/etc/wpa_supplicant/wpa_supplicant.conf`.  The contents depend on whether your home network is open, WEP or WPA.  It is
probably WPA, and so should look like:

    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    country=GB
    
    network={
	    ssid="_ST_SSID_"
	    scan_ssid=1
	    psk="_ST_PASSWORD_"
	    key_mgmt=WPA-PSK
    }

Replace `_ST_SSID_` with your home network SSID and `_ST_PASSWORD_` with your wifi password (in clear text).

Now to activate this, run the following command:

    wpa_cli reconfigure

## Install the packages you need for DNS, Access Point and Firewall rules.

    apt-get update
	apt-get install hostapd dnsmasq iptables-persistent

## /etc/dnsmasq.conf

    interface=lo,uap0
    no-dhcp-interface=lo,wlan0
    bind-interfaces
    server=8.8.8.8
    dhcp-range=10.3.141.50,10.3.141.255,12h

## /etc/hostapd/hostapd.conf

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

    DAEMON_CONF="/etc/hostapd/hostapd.conf"

## Now restart the dns and hostapd services

    systemctl restart dnsmasq
    systemctl restart hostapd

## All done or...
At this point, I expect everything to work after a reboot. Instead, I end up with wlan0 UP and ap0 DOWN (or vice-versa) when checking via ip addr, and dmesg indicates some errors in the Broadcom driver. After some tinkering, I discovered the following sequence of commands after a reboot would get both interfaces up and working simultaneously, like we wanted all along:
    
    sudo ifdown --force wlan0
    sudo ifup wlan0
    
That's, all.

