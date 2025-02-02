# Configuring DNS for Raspbian

There are many variations on this theme. This tutorial covers:

- [Automatic DNS configuration](#autoDNS)
- [Static DNS configurations](#staticDNS)

	- [DNS server is a different host](#otherhostDNS)
	- [DNS server is this Raspberry Pi](#localhostDNS)

<hr>

## <a name="autoDNS"> Automatic DNS configuration </a>

Out of the box, a fresh Raspberry Pi OS installation assumes that a DHCP server will provide:

* IP address(es) for the Raspberry Pi's Ethernet and/or WiFi interfaces; and
* IP address(es) of DNS server(s).

There is nothing you need to do on the Raspberry Pi to make this work. All configuration needs to occur on the DHCP server which, in most home networks, is your router.

Running IOTstack implies your Raspberry Pi will be offering *services*. That means it will be operating as a *server*. Servers need *predictable* IP addresses.

The simplest way to achieve this is through a static binding in your DHCP server.

To set up static bindings you need the Media Access Control (MAC) address of each network interface. Depending on whether you want to set up your Ethernet or WiFi interfaces, or both, you will need either or both of the following:

```bash
$ ifconfig eth0
$ ifconfig wlan0
```

In each case, look for the line beginning with `ether`.

Once you have set up the static binding in your DHCP server, reboot your Raspberry Pi and re-run the `ifconfig` commands to make sure the Pi has picked up the addresses you assigned.

You can confirm that your Raspberry Pi is using the expected DNS server(s) by running:

```bash
$ cat /etc/resolv.conf
```

## <a name="staticDNS"> Static DNS configurations </a>

### <a name="otherhostDNS"> DNS server is a different host </a>

Make the following assumptions:

1. You have a DNS server running on a local host with the IP address 192.168.203.50.
2. Your local DNS server is authoritative for the domain `my.domain.com`.
3. There is a good reason why you can't configure your DHCP server to provide either or both of those parameters to DHCP clients.

Proceed like this:
 
1. Move to the correct directory:

	```bash
	$ cd /etc
	```

2. Make a backup copy of `resolvconf.conf`:

	```bash
	$ sudo cp resolvconf.conf resolvconf.conf.bak
	```

3. Use `sudo` and the text editor of your choice to edit `resolvconf.conf`. Find these lines:

	```
	# configure your subscribers configuration files below.
	#name_servers=127.0.0.1
	```

4. After those lines, insert the IP address of your local upstream DNS server. For example:

	```
	name_servers=192.168.203.50
	search_domains=my.domain.com
	```

	Notes:

	* If you do **not** have a local domain, omit the `search_domains=` line.
	* If you **do** have a local domain, you may find that your DHCP server:

		- *can* supply the IP address(es) of your DNS servers (eg 192.168.203.50), but
		- *can't* supply your local domain name.

		This is a common pattern with many home routers. Just omit the `name_servers=` line (so the DNS servers come from DHCP) while retaining the `search_domains=` line (so the domain is configured statically).

	* If you need to configure multiple static DNS servers, use space-separated notation and encapsulate the right hand side in quotes. For example:

		```
		name_servers="192.168.203.50 192.168.203.51"
		```

5. Save your work and restart the DHCP client service:

	```bash
	$ sudo service dhcpcd reload
	$ sudo resolvconf -u
	```

6. Check the result (the file in the following is not a typo):

	```bash
	$ cat resolv.conf
	```

	Given the above settings, you would expect to see:

	```
	# Generated by resolvconf
	search my.domain.com
	nameserver 192.168.203.50
	```

7. Prepare a patch file.

	* If the configuration embedded in `resolvconf.conf` is sufficiently *general* to be useful for all of your Raspberry Pi hosts, run:

		```bash
		$ diff resolvconf.conf.bak resolvconf.conf >~/resolvconf.conf.patch
		```

	* If the configuration embedded in `resolvconf.conf` is unique to **this** particular Raspberry Pi, run:

		```bash
		$ diff resolvconf.conf.bak resolvconf.conf >~/resolvconf.conf.patch@$HOSTNAME
		```

8. Move the patch file to the folder:

	```
	~/PiBuilder/boot/scripts/support/etc/
	```

	The next time you build a Raspberry Pi using PiBuilder, your resolver configuration will be set automatically.

### <a name="localhostDNS"> DNS server is this Raspberry Pi </a>

This is appropriate if your Raspberry Pi is running a DNS server (eg BIND9), either as a native install or in a container running in Host Mode.

You *can* use PiHole for a local DNS but you may run into problems if it does not start early enough for the other containers you are running.

Follow [DNS server is a different host](#otherhostDNS) but, instead of adding a `name_servers` line in step 4, simply uncomment the line:

```
#name_servers=127.0.0.1
```

## <a name="baselineReference"> Reference versions of files </a>

At the time of writing (November 2021), these were the baseline versions of `/etc/resolvconf.conf` on Buster and Bullseye.

### `/etc/resolvconf.conf` - Raspbian Buster

```
# Configuration for resolvconf(8)
# See resolvconf.conf(5) for details

resolv_conf=/etc/resolv.conf
# If you run a local name server, you should uncomment the below line and
# configure your subscribers configuration files below.
#name_servers=127.0.0.1

# Mirror the Debian package defaults for the below resolvers
# so that resolvconf integrates seemlessly.
dnsmasq_resolv=/var/run/dnsmasq/resolv.conf
pdnsd_conf=/etc/pdnsd.conf
unbound_conf=/var/cache/unbound/resolvconf_resolvers.conf
```

### `/etc/resolvconf.conf` - Raspbian Bullseye

```
# Configuration for resolvconf(8)
# See resolvconf.conf(5) for details

resolv_conf=/etc/resolv.conf
# If you run a local name server, you should uncomment the below line and
# configure your subscribers configuration files below.
#name_servers=127.0.0.1


# Mirror the Debian package defaults for the below resolvers
# so that resolvconf integrates seemlessly.
dnsmasq_resolv=/var/run/dnsmasq/resolv.conf
pdnsd_conf=/etc/pdnsd.conf
unbound_conf=/etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
```

### Summary of differences

The differences between the Bullseye and Buster versions are:

1. The Bullseye version has an additional blank line before `# Mirror …`; and
2. The last line (`unbound_conf`) has a different path.

As long as you confine your changes to the area before the additional blank line, a patch prepared on Buster should also work on Bullseye.
