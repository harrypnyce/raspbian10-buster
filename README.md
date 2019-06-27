Raspbian GNU/Linux 10 (buster) Lite setup: (Wireguard, Pi-hole & Unbound)

> Linux pihole2 4.19.56-v7+ #1242 SMP Wed Jun 26 17:31:47 BST 2019 armv7l GNU/Linux

Raspbian Buster Lite initial install.		
Change default username & password (pi/raspberry).

    passwd
    usermod -aG sudo user
    groupadd user
    usermod -d /home/user -m -g user -l user pi
    sudo rpi-update

Use console based raspi-config application to make configuration changes.

    sudo raspi-config
    
Generate SSH key pairs.

    ssh-keygen -t rsa -b 4096

[Optional]

    ssh-copy-id -i ~/.ssh/id_rsa user@host/ip

[Optional] Take some to configure and harden your SSH clients and servers.

    nano /etc/ssh/sshd_config
    sudo nano .bash_aliases
    sudo apt update && sudo apt-get upgrade -y

[Optional] Add real-time clock DS3231 to RPi3 B+.

    apt-get purge fake-hwclock
    sudo apt-get install python-smbus i2c-tools
    sudo nano /etc/modules
		rtc-ds1307
    sudo i2cdetect -y 1
    sudo nano /etc/rc.local
		"echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
    hwclock -s"
    sudo reboot
    date
    sudo date -s "Thu 27 Jun 2019 01:41:20"
    sudo hwclock -w
    date; sudo hwclock -r

    sudo apt-get install hostapd dnsmasq libmnl-dev linux-headers-rpi build-essential git dnsutils bc raspberrypi-kernel-headers iptables-persistent -y
    echo "deb http://deb.debian.org/debian/ unstable main" > /etc/apt/sources.list.d/unstable-wireguard.list
    sudo su
    printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' > /etc/apt/preferences.d/limit-unstable
    sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 7638D0442B90D010
    sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC
    sudo apt update
    sudo apt-get install wireguard -y
    sudo reboot		

Enable IPv4 forwarding (reboot required to activate forwarding).

    sudo perl -pi -e 's/#{1,}?net.ipv4.ip_forward ?= ?(0|1)/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf
		sudo reboot

Confirm previous changes.

    sysctl net.ipv4.ip_forward		
		
Configure WireGuard

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


    "[Interface]
    Address = 192.168.99.1/24
    ListenPort = 51820

    PrivateKey = <server_private.key>
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

    [Peer]
    #Client1
    PublicKey = <client1_public.key>
    AllowedIPs = 192.168.99.2/32"

Start Wireguard VPN server

    sudo wg-quick up wg0		
    sudo wg		

Automatically launch Wireguard at system startup.		

    sudo systemctl enable wg-quick@wg0
    sudo apt install qrencode
		
Install Pi-hole (select upstream DNS servers)... we will be changing this to use Unbound.

    sudo curl -sSL https://install.pi-hole.net | bash
