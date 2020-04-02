# Self hosted DNS project 

## Description
To learn more about Linux administration as well as DNS and BIND I wanted to host my own DNS to block ads and other malicious websites, and also host my own internal sites with DNS resolution.

## Setup

### Server installation

* Install a CentOS minimal installation.
* Create a user for me.

```
# adduser marcus
# passwd marcus
```
* Set up public key authentication for my user.
  * On my PC, generate a RSA key pair.

```
$ ssh-keygen
```
  * Copy public key to the CentOS server.
* For security reasons, I disable root login via ssh.

```
Edit /etc/ssh/sshd_config:
	PermitRootLogin no
	PasswordAuthentication no
```
* Set a static IP address.

```
Edit /etc/sysconfig/network-scripts/INTERFACE:

....
TYPE=Ethernet
IPADDR=192.168.0.100
PREFIX=24
GATEWAY=192.168.0.1
DNS1=192.168.0.100
....
```

### BIND

* Install bind and bind-utils

```
# yum install bind bind-utils
```

* Create zone files for my own internal zone and reverse zone for 0.168.192.in-addr.arpa.

```
/var/named/eccologic.se.zone

$TTL 86400
@ IN SOA dns-1.eccologic.se. dns-1.eccologic.se. (
        2019020201 ; Serial
        1d ; refresh
        2h ; retry
        4w ; expire
        1h ); min cashe

    IN      NS      dns-1.eccologic.se.

dns-1           IN      A       192.168.0.100

/var/named/eccologic.se.revzone

$TTL 86400
@ IN SOA dns-1.eccologic.se. dns-1.eccologic.se. (
        2019020201 ; Serial
        1d ; refresh
        2h ; retry
        4w ; expire
        1h ); min cashe

    IN      NS      dns-1.eccologic.se.

100             IN      PTR     dns-1.eccologic.se.
```
* Some BIND configuration in /etc/named.conf

  * Edit configuration to allow server to listen to queries from any address

  * Add forwarders to my ISP's DNS servers

  * Change directory to match with where I have placed my zone files

  * Added zones for my own internal zone and reverse zone
 
```
options {

	....
	....
	allow-query { any; };
	forwarders { 83.255.255.1; 83.255.255.2; };
	directory "/var/named";
	....
	....

};

zone "eccologic.se" {
        type master;
        file "eccologic.se.zone";
        allow-update { none; };
};


zone "0.168.192.in-addr.arpa" {
        type master;
        file "eccologic.se.revzone";
        allow-update { none; };
};
```

* Enable named and start it

```
# systemctl enable named
# systemctl start named
```
* Verify that lookup works in my own internal zone and in external zones (not hosted by my DNS)

```
$ dig eccologic.se @192.168.0.100

;; AUTHORITY SECTION:
eccologic.se.             3600    IN      SOA     dns-1.eccologic.se. dns-1.eccologic.se. 2019020201 86400 7200 2419200 3600

$ dig google.com @192.168.0.100

;; ANSWER SECTION:
google.com.             237     IN      A       172.217.22.174
```
### Adblocking

* Download the preferred list of malicious hosts from https://github.com/StevenBlack/hosts and save it to a file named adblock.zones

```
$ wget https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts -O adblock.zones
```
* Modify the list to fit the BIND zone file formatting

```
$ cat adblock.hosts | grep '^0.0.0.0' | egrep -v '127.0.0.1|255.255.255.255|::1' | cut -d " " -f 2 >> adblock.temp
$ cat adblock.temp | egrep -v '^$|#' | sort | uniq > adblock.hosts
$ cat adblock.hosts | sed -r 's/(.*)/zone "\1" {type master; file "null.zone.file";};/' > adblock.zone
# cp adblock.zone /var/named/
$ rm -rf adblock.hosts adblock.temp
```
* Create a zone file called null.zone.file, which should point to your DNS server

```
$TTL 86400
@ IN SOA dns-1.eccologic.se. dns-1.eccologic.se. (
        2019012702 ; Serial
        1d ; refresh
        2h ; retry
        4w ; expire
        1h ); min cashe
        IN      NS      dns-1.eccologic.se.
        IN      A       192.168.0.100

@       IN      A       192.168.0.100
*       IN      A       192.168.0.100
```
* Add the adblock zone to BIND config

```
Edit /etc/named.conf

include "/var/named/adblock.zone"
```
* Restart named service

```
# systemctl restart named
```
### Website

* Install necessary packages 

```
# yum install http openssl mod_ssl
```

* Allow firewalld to allow http and https

```
# firewall-cmd --permanent --add-service={http,https}
```
* Generate private key and self-signed cert

```
# openssl genrsa -out ca.key 2048
# openssl req -new -key ca.key -out ca.csr
# openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```
* Move key and certificate

```
# mv ca.crt /etc/pki/tls/certs
# mv ca.key /etc/pki/tls/private/ca.key
```
* Edit Apache SSL configuration file to the correct path of the certificate
```
Edit /etc/httpd/conf.d/ssl.conf

....
SSLCertificateFile /etc/pki/tls/certs/ca.crt
SSLCertificateKeyFile /etc/pki/tls/private/ca.key
....
```
* Create a custom html site under /var/www/html/ named index.html
```
# echo '<h1>Site blocked by DNS</h1><br><p>Contact your system administrator if you feel that this site should be unblocked!</p>' > /var/www/html/index.html
```
* Enable and start Apache

```
# systemctl enable httpd
# systemctl start httpd
```
* Verify that you can reach the site, and that you can see your index.html site

### Final

* Make sure you change your specified DNS server, either on your router or your local PC. And verify through dig/nslookup/ping that your DNS resolves an address from the blocklist to itself.

```
$ cat /etc/resolv.conf
nameserver 192.168.0.100

$ dig ads.contentabc.com

....
....

;; ANSWER SECTION:
ads.contentabc.com.     86400   IN      A       192.168.0.100

;; AUTHORITY SECTION:
ads.contentabc.com.     86400   IN      NS      dns-1.eccologic.se.
```

* As we can see, the resolution works! Mission complete!
