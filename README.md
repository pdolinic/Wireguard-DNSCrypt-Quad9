# Wireguard-Dnscrypt-Quad9 Install-Guide
Installation &amp; Konfiguration von Wireguard mit Dnscrypt &amp; Quad 9

Vorteile: https://www.ivpn.net/pptp-vs-ipsec-ikev2-vs-openvpn-vs-wireguard/

# Blogpost https://www.netways.de/blog/2021/03/18/lust-auf-mehr-privacy-diy-wireguard-mit-dnscrypt-quad9

# Hinweise

--+ Hinweis1 Quote bzgl Pre-Shared-Key---


Pre-Shared Key as additional security #extra security

The connection can optionally also be further secured by using an additional pre-shared key.[4]

You can easily create a pre-shared key with the tool wg:

    wg genpsk > presharedkey

Then add the following line to the [Peers] section of the WireGuard configuration, in this example wg0.conf.

    Presharedkey = <Pre-Shared Key>
    
    
--# Hinweis1 Quote  bzgl Pre-Shared-Key---


--+ Hinweis2---

X.X.X.X ersetzen durch gewünschtes lokales IP-Netz des Wireguard adapter, z.B. 192.168.0.X etc. https://en.wikipedia.org/wiki/Private_network

--# Hinweis2---


# Installations & Konfigurations Start, bitte direkt unter Root arbeiten via

    sudo -i
 

# Server-Seite A

Auf den neuesten Stand bringen

    dnf update

Einbinden des benötigten Repos:

    dnf install -y elrepo-release epel-release

Kernel-Modle installieren falls nicht geschehen:

    dnf install kmod-wireguard wireguard-tools

Verzeichnis anlegen:

    mkdir -v /etc/wireguard/

Wechseln:

    cd /etc/wireguard

Konfig anlegen:

    sh -c 'umask 077; touch /etc/wireguard/wg0.conf'

Public & Private Key generieren:

    sh -c 'umask 077; wg genkey | tee privatekey | wg pubkey > publickey'

Private & Public keys notieren:

    cat publickey && cat privatekey
    
    

Interface editieren unter  /etc/wireguard/wg0.conf
Interface wg0 Konfiguration:

---+ /etc/wireguard/wg0.conf Server

    [Interface]

VPN server private IP address ##

    Address = X.X.X.1/24 #lokaleIP extra für den WG0Adapter anlegen
 
Save and update this config file when a new peer (vpn client) added ##

    SaveConfig = true
 
VPN server port ##

    ListenPort = 30000
 
    PrivateKey = <Private_Key_Server> #zuvor generierter Schlüssel
 
    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    
    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

---# /etc/wireguard/wg0.conf Server



Wireguard Firewall Regeln generieren:

    vi /etc/firewalld/services/wireguard.xml

Firewalld-Konfig unter (RHEL /usr/lib/firewalld/services/ ) ( Centos : /etc/firewalld/services/wireguard.xml )

---+ /etc/firewalld/services/wireguard.xml

    <?xml version="1.0" encoding="utf-8"?>
    <service>
    <short>wireguard</short>
    <description>WireGuard open UDP port 30000 for client connections</description>
    <port protocol="udp" port="30000"/>
    </service>

---# /etc/firewalld/services/wireguard.xml



 Firewall regeln aktivieren:

    firewall-cmd --permanent --add-service=wireguard --zone=public

Masqueading auf Public aktivieren:

    firewall-cmd --permanent --zone=public --add-masquerade

Firewall neustarten & prüfen:

    firewall-cmd --reload && sudo firewall-cmd --list-all
    

Centos benötigt zusätzliche Forwarding Parameter die unter /etc/sysctl.d/99-custom.conf gesetzt werden:

---+ /etc/sysctl.d/99-custom.conf 

Turn on bbr ##

    net.core.default_qdisc = fq
    net.ipv4.tcp_congestion_control = bbr
  
for IPv4 <-> Muss gesetzt sein##

    net.ipv4.ip_forward = 1
  
Turn on basic protection/security ##

    net.ipv4.conf.default.rp_filter = 1
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.tcp_syncookies = 1
  
Fuer Ipv6, besser mal anlassen da DNSCrypt per Default auf Ipv6 zu arbeiten schien: ##

    net.ipv6.conf.all.forwarding = 1
    
Falls du IPV6 abschalten willst, ansonsten ist die Konfiguration an dieser Stelle fertig:

    #net.ipv6.conf.all.disable_ipv6 = 1
    #net.ipv6.conf.default.disable_ipv6 = 1

---# /etc/sysctl.d/99-custom.conf 


    
Zum direkten aktivieren ohne Neustart:

    sysctl -p /etc/sysctl.d/99-custom.conf

Wg0 interface anschalten & Masquerading auf Internal setzen:

    firewall-cmd --add-interface=wg0 --zone=internal --permanent
    firewall-cmd --permanent --zone=internal --add-masquerade


Services starten (Pro Änderung,z.B. neuer Client, immer WG-Service stoppen, Client eintragen, Service starten):

    systemctl enable wg-quick@wg0.service && sudo systemctl start wg-quick@wg0 && sudo systemctl status wg-quick@wg0

Check (optional)

    sudo wg

 

 
# Client-Seite

 
Tools installieren

    dnf install wireguard-tools

Verzeichnis anlegen und Konfigdatei erstellen

    mkdir -v /etc/wireguard/
    sh -c 'umask 077; touch /etc/wireguard/wg0.conf'

Keys generieren:

    sh -c 'umask 077; wg genkey | tee privatekey | wg pubkey > publickey'

Public & Privatekey auslesen

    cat privatekey; cat publickey

Die wg0 .conf sollte folgendes beinhalten:
    
---+ wg0.conf - Client 

    [Interface]

client private key ##
        
    PrivateKey = PrivateKeyClient
  
client ip address ##

    Address = X.X.X.2/24 #eigene ip dieses rechners im server-subnet
  
    [Peer]

CentOS 8 server public key ##

    PublicKey = ServerPublicKey
  


Turn on NAT for client so internet routed thorugh our vpn

    AllowedIPs = 0.0.0.0/0
  
Your CentOS 8 server's public IPv4/IPv6 address and port ##

    Endpoint = PublicServerIP:Port
  
Key connection alive ##

    PersistentKeepalive = 15

---# wg0.conf - Client 


Services starten & testen, danach stoppen da noch keine Verbindung verfügbar ist:

    systemctl enable wg-quick@wg0; sudo systemctl start wg-quick@wg0; sudo systemctl status wg-quick@wg0;sudo systemctl start wg-quick@wg0;

 
# Server Seite B

    systemctl stop wg-quick@wg0

    vi /etc/wireguard/wg0.conf
    

Nachtragen des Public-Key des Clients & der Client-IP-Zuweisung unter wg0.conf:

---+ wg0.conf - Server

    [Peer]
    
client VPN public key ##

    PublicKey = PublicKeyClient
  
client VPN IP address (note /32 subnet) ##

    AllowedIPs = X.X.X.2/32 #Achtung /32-beachten muss hier gesetzt werden

---# wg0.conf - Server


Neustarten

    systemctl start wg-quick@wg0;



# DNS-Crypt

Installieren

    dnf install dnscrypt-proxy

Konfiguration unter /etc/dnscrypt-proxy/dnscrypt-proxy.toml

---+ /etc/dnscrypt-proxy/dnscrypt-proxy.toml
    
    server_names = ['quad9-dnscrypt-ip4-filter-pri'] #nach belieben anpassen https://dnscrypt.info/public-servers/
    listen_addresses = ['0.0.0.0:53'] #AlleInterfaces incl Wireguard
    ipv4_servers = true
    ipv6_servers = false
    dnscrypt_servers = true
    doh_servers = true
    require_dnssec = true
    require_nolog = true
    require_nofilter = false
    fallback_resolvers = ['9.9.9.9:53', '8.8.8.8:53']
     
     
    [sources.quad9-resolvers]
    urls = ['https://www.quad9.net/quad9-resolvers.md']
    minisign_key = 'RWQBphd2+f6eiAqBsvDZEBXBGHQBJfeG6G+wJPPKxCZMoEQYpmoysKUN'
    cache_file = 'quad9-resolvers.md'
    prefix = 'quad9-'

---# /etc/dnscrypt-proxy/dnscrypt-proxy.toml

 Anschalten:

    systemctl enable dnscrypt-proxy;systemctl start dnscrypt-proxy;systemctl status dnscrypt-proxy

Nun muss unter /etc/resolv.conf die alten Resolver abstellen und auf localhost umstellen, dazu einfach am besten direkt in der Sysconfig unter: /etc/sysconfig/network-scripts/adapter:

DNS1 auf localhost setzen, und DNS2 loeschen:

    DNS1="127.0.0.1"     


NetzwerkManager neustarten:

    service NetworkManager restart

Test:

    cat /etc/resolf.conf;dig google.de

 
# IPTables / Firewalld für Wireguard mit DNSCrypt

Folgende Regeln sollten bereits in der /etc/wireguard/wg0.conf gesetzt worden sein:

    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

---+ neue IPTables Regeln (funktioniert unter Centos da iptables-translate aktiv ist)
      
     iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
     iptables -A INPUT -s X.X.X.X/24 -p udp -m udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
     iptables -A INPUT -s 127.0.0.1/32 -p udp -m udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
     iptables -A INPUT -i lo -j ACCEPT
     iptables -A INPUT -p udp -m udp --dport 30000 -m conntrack --ctstate NEW -j ACCEPT
     iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
     iptables -A FORWARD -i wg0 -j ACCEPT
     iptables -A OUTPUT -o lo -j ACCEPT
     iptables -P INPUT DROP
     iptables -P FORWARD DROP
     iptables -P OUTPUT ACCEPT

---# neue IPTables Regeln (funktioniert unter Centos da iptables-translate aktiv ist)

Abspeichern (unter Centos: /etc/syconfig/iptables):

    iptables-save > /etc/sysconfig/iptables-config

DNS auf Internal zulassen:

    firewall-cmd --add-service=dns --zone=internal --permanent

Rich rule auf Public setzen:

    firewall-cmd --zone=public --permanent --add-rich-rule='rule family="ipv4" source address="X.X.X.X/24" accept'

Firewalld neustarten:

    firewall-cmd --reload && firewall-cmd --list-all-zones

Finaler Netzwerk Neustart:
    
    service Networkmanager restart


Quellen:


https://github.com/BetterWayElectronics/secure-wireguard-implementation # Guide Top - Main

https://www.cyberciti.biz/faq/centos-8-set-up-wireguard-vpn-server/ #Guide Top - Main

https://www.suse.com/c/pihole-podman-opensuse/ #Suse podman

https://linuxtips.us/install-wireguard-centos-8/ #Guide2

https://gist.github.com/Anachron/e2ba7ace4e4ef6988182adc7462ffb80 #Tutorial Ubuntu

https://tau.gr/posts/2019-03-03-set-up-cloudflared-ubuntu-wireguard/ #Wireguard mit Cloudflared-Resolver

https://www.linuxbabe.com/centos/wireguard-vpn-server-centos # Guide 2# Centos & Wireguard

https://www.educba.com/dns-configuration-in-linux/

https://linuxize.com/post/install-configure-fail2ban-on-centos-8/

https://stanislas.blog/2019/01/how-to-setup-vpn-server-wireguard-nat-ipv6/

https://github.com/notasausage/pi-hole-unbound-wireguard/blob/master/wg0.conf # Wireguard Pihole Unbound

 https://github.com/linuxserver/docker-wireguard/
 
https://notes.iopush.net/blog/2020/wireguard-and-pi-hole-in-docker/

https://serverfault.com/questions/626521/centos-7-save-iptables-settings #Iptables speichern

https://scotthelme.co.uk/securing-dns-across-all-of-my-devices-with-pihole-dns-over-https-1-1-1-1

https://wiki.archlinux.org/index.php/Dnscrypt-proxy

https://www.rahulpandit.com/post/wireguard-vpn-search-privately-searx-block-ads-and-tracking-pihole-dnscrypt-proxy/#add-the-missing-route #Hinweis Routing

https://freedombox.org/ # All-in-One-Alternative mit OpenVPN, Matrix, Search uvm.

https://gitlab.com/cyber5k/mistborn # All-in-One-Alternative, Interessantes Komplettpacket

https://github.com/IAmStoxe/wirehole #Docker Komplettlösung unter Ubuntu


