[Raspbian GNU/Linux 10 (buster) Lite](https://downloads.raspberrypi.org/raspbian_lite_latest.torrent) setup: (Wireguard, Pi-hole & Unbound) running on a Raspberry Pi 3 B+

> ~~Linux pihole2 4.19.56-v7+ #1242 SMP Wed Jun 26 17:31:47 BST 2019 armv7l GNU/Linux~~ (updated for new kernel)
> Linux pihole2 4.19.75-v7+ #1271 SMP Mon Sep 30 13:49:02 BST 2019 armv7l GNU/Linux

Raspbian Buster Lite initial install.		
[Change default username](https://unix.stackexchange.com/questions/98461/proper-way-of-changing-username-in-ubuntu-or-any-linux) & password (pi/raspberry).

    passwd
    usermod -aG sudo user
    groupadd user
    usermod -d /home/user -m -g user -l user pi 
~~sudo rpi-update~~ [latest bleeding-edge firmware and kernel no longer needed for these use-case(s)] https://github.com/Hexxeh/rpi-update

[Use console based raspi-config](https://www.raspberrypi.org/documentation/configuration/raspi-config.md) application to make configuration changes.

    sudo raspi-config
    
[Generate SSH key pairs](https://www.ssh.com/ssh/keygen/).

    ssh-keygen -t rsa -b 4096

[Optional]

    ssh-copy-id -i ~/.ssh/id_rsa user@host/ip

[Optional] Take some time to [configure and harden your SSH server](https://infosec.mozilla.org/guidelines/openssh.html).

    nano /etc/ssh/sshd_config
    sudo apt update && sudo apt-get upgrade -y

[Optional] [Add real-time clock DS3231](https://sigmdel.ca/michel/ha/rpi/rtc_en.html) to RPi3 B+ (for DNSSEC accuracy, as the Raspberry Pi devices lack a proper hardware clock).

https://www.raspberrypi-spy.co.uk/2015/05/adding-a-ds3231-real-time-clock-to-the-raspberry-pi/

    apt-get purge fake-hwclock
    sudo apt-get install python-smbus i2c-tools
    sudo nano /etc/modules
		rtc-ds1307
    sudo i2cdetect -y 1
    sudo nano /etc/rc.local
		echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
    		hwclock -s
    sudo reboot
    date
    sudo date -s "Thu 27 Jun 2019 01:41:20"
    sudo hwclock -w
    date; sudo hwclock -r

[Wireguard installation](https://pivpn.dev/) (original notes based off of this great script --> https://github.com/adrianmihalko/raspberrypiwireguard).
    curl -L https://install.pivpn.dev | bash
The above Wireguard installer handles EVERYTHING -- once finished, please reboot and skip down to the "Unbound" section and proceed normally.
    
If you can do this better yourself, please continue, HOWEVER i urge you to consider utilizing the superb installer from team at www.pivpn.dev as it allows for customization of VPN port, encryption strength, DNS server, etc.  It's extremely powerful, even for experts -- also allows option of using OpenVPN server installation if you're not yet ready to try Wireguard.

    sudo apt-get install hostapd dnsmasq libmnl-dev linux-headers-rpi build-essential git dnsutils bc raspberrypi-kernel-headers iptables-persistent -y
    echo "deb http://deb.debian.org/debian/ unstable main" > /etc/apt/sources.list.d/unstable-wireguard.list
    sudo su
    printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' > /etc/apt/preferences.d/limit-unstable
    sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC
    
~~sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC~~ (_only the first pubkey was necessary_)

    sudo apt update
    sudo apt-get install wireguard -y
    sudo reboot		

Enable IPv4 forwarding (reboot required to activate forwarding).

    sudo perl -pi -e 's/#{1,}?net.ipv4.ip_forward ?= ?(0|1)/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf
    sudo reboot

Confirm previous changes.

    sysctl net.ipv4.ip_forward		
		
[Configure WireGuard](https://github.com/adrianmihalko/raspberrypiwireguard)

    mkdir wgkeys		
    cd wgkeys		

PLEASE protect your private keys!

    wg genkey > server_private.key		
    wg pubkey > server_public.key < server_private.key		
    wg genkey > client1_private.key		
    wg pubkey > client1_public.key < client1_private.key		
    ls -la		
    cat server_private.key		
    cat client1_public.key

Setup Wireguard VPN server network interface, using server PRIVATE key & client PUBLIC key.

    sudo nano /etc/wireguard/wg0.conf		


    [Interface]
    Address = 192.168.99.1/24
    ListenPort = 51820

    PrivateKey = <server_private.key>
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

    [Peer]
    #Client1
    PublicKey = <client1_public.key>
    AllowedIPs = 192.168.99.2/32

Start Wireguard VPN server

    sudo wg-quick up wg0		
    sudo wg		

Automatically launch Wireguard at system startup.		

    sudo systemctl enable wg-quick@wg0
    sudo apt install qrencode
		
Install [Pi-hole then Unbound](https://docs.pi-hole.net/guides/unbound/).

    sudo curl -sSL https://install.pi-hole.net | bash

Setting up Pi-hole as a recursive DNS server.

	sudo apt install unbound
	
Download current root hints file.

	wget -O root.hints https://www.internic.net/domain/named.root
	sudo mv root.hints /var/lib/unbound/
	
Configure unbound

	sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
	
    server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 2

    port: 5353
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the servers authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # Suggested by the unbound man page to reduce fragmentation reassembly problems
    edns-buffer-size: 1472

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes
    
    # This attempts to reduce latency by serving the outdated record before
    # updating it instead of the other way around. Alternative is to increase
    # cache-min-ttl to e.g. 3600.
    cache-min-ttl: 0
    serve-expired: yes
    # serve-expired-ttl: 3600 # 0 or not set means unlimited (I think)

    # Use about 2x more for rrset cache, total memory use is about 2-2.5x
    # total cache size. Current setting is way overkill for a small network.
    # Judging from my used cache size you can get away with 8/16 and still
    # have lots of room, but I've got the ram and I'm not using it on anything else.
    # Default is 4m/4m
    msg-cache-size: 128m
    rrset-cache-size: 256m

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
    
Start your local recursive DNS server (and test).

	sudo service unbound start
	dig pi-hole.net @127.0.0.1 -p 5353

~~Create symbolic link to fix potential lighttpd breakage~~ [issues with Pi-hole on Debian Buster](https://github.com/pi-hole/pi-hole/issues/2557).
*** This can be skipped, as it should now be resolved in the Latest Pi-hole v4.3.1 update. (Saturday, June 29, 2019) ***

*[SKIP THIS PORTION (regarding lighttpd)]*
	
	cd /usr/share/lighttpd/
	sudo ln -s create-mime.conf.pl create-mime.assign.pl

Wireguard Routing, NAT and Firewall

	sudo iptables -A INPUT -i eth0 -p tcp --dport 80 -j ACCEPT
	sudo iptables -A INPUT -i eth0 -p tcp --dport 53 -j ACCEPT
	sudo iptables -A INPUT -i eth0 -p udp --dport 53 -j ACCEPT
	sudo iptables -A INPUT -i eth0 -p udp --dport 67 -j ACCEPT
	sudo iptables -A INPUT -i eth0 -p udp --dport 68 -j ACCEPT
	sudo netfilter-persistent save
	sudo netfilter-persistent reload

Enable NAT

	sudo iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
	
Allow any traffic from eth0 (internal) to go over wg0 (point to point tunnel)

	sudo iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT

Allow RELATED, ESTABLISHED wg0 (point to point tunnel) traffic to (internal) eth0 network

	sudo iptables -A FORWARD -i wg0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT

	sudo iptables -A INPUT -i lo -j ACCEPT

	sudo iptables -A INPUT -i eth0 -p icmp -j ACCEPT

	sudo iptables -A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT

	sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

	sudo iptables -P FORWARD DROP
	sudo iptables -P INPUT DROP
	sudo iptables -L

	sudo systemctl enable netfilter-persistent
	sudo netfilter-persistent reload
	
[Convert IPtables to nftables](https://wiki.nftables.org/wiki-nftables/index.php/Moving_from_iptables_to_nftables) (v0.9.0-2)
	
	sudo apt install nftables

Any untranslated rule(s) will be prefixed by a hash sign (#), as shown in the following example:

	iptables-translate -A INPUT -j CHECKSUM --checksum-fill
nft # -A INPUT -j CHECKSUM --checksum-fill

	iptables-save > rules.iptables
	iptables-restore-translate -f rules.iptables > rules.nft
	nft -f rules.nft
	nft list ruleset
