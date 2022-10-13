# Introduction
In this repo a simple instruction on how connecting Cisco OpenConnect servers to each other is provided. Assume the server A is located in Iran and its ip is 1.0.0.0 with Ethernet interface eth0 on 1.0.0.1. The other server B is located in any other country with ip 2.0.0.0, eth0 on 2.0.0.1. The user's ip is considered to be 3.0.0.0. We assume both servers run Centos 7 or 8, but the overal method is the same for other Linux distributions.

# 1- Installing OpenConnect Server on server B
Login to your server directly or using ssh.
## 1-1 Install epel repository:
```
yum install epel-release
```
## 1-2 Update Centos
```
yum update
```
## 1-3 Install ocserv
```
yum install ocserv gnutls-utils
```
## 1-4 Create cert directory
```
mkdir /etc/ocserv/cert
cd /etc/ocserv/cert/
```
## 1-5 Create a template file
```
touch ca.tmpl
yum install nano
nano ca.tmpl
```
Write these contents in the file:
```
cn = "VPN CA"
organization = "Big Corp"
serial = 1
expiration_days = -1
ca
signing_key
cert_signing_key
crl_signing_key
```
## 1-6 Generate certificates
```
certtool --generate-privkey --outfile ca-key.pem
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```
## 1-7 Create a server template
```
nano /etc/ocserv/server.tmpl
```
Write these contents in the template:
```
cn = "My server"
dns_name = "www.example.com"
organization = "MyCompany"
expiration_days = -1
signing_key
encryption_key
tls_www_server
```
## 1-8 Issue private key and certificate
```
certtool --generate-privkey --outfile server-key.pem
certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template /etc/ocserv/server.tmpl --outfile server-cert.pem
```
## 1-9 Create the ssl directory
```
mkdir /etc/ocserv/ssl/
```
## 1-10 Copy the certificate files to the ssl directory
```
cp ca-cert.pem server-key.pem server-cert.pem /etc/ocserv/ssl/
```
## 1-11 Config the ocserv
```
nano /etc/ocserv/ocserv.conf
```
find and edit the following settings:
```
auth = "plain[passwd=/etc/ocserv/passwd]"
server-cert = /etc/ocserv/ssl/server-cert.pem
server-key = /etc/ocserv/ssl/server-key.pem
ca-cert = /etc/ocserv/ssl/ca-cert.pem
ipv4-network = 192.168.1.0/24
```
Add the following lines to the file:
```
dns = 8.8.8.8
dns = 4.2.2.4
```
## 1-12 Install CSF
### 1-12-1 Disable firewalld
```
systemctl disable firewalld
systemctl stop firewalld
```
### 1-12-2 Install iptable-services
```
yum install iptables-services
```
### 1-12-3 Create config files for iptables
```
touch /etc/sysconfig/iptables
touch /etc/sysconfig/ip6tables
```
### 1-12-4 Start iptables
```
systemctl start iptables
systemctl start ip6tables
```
### 1-12-5 Enable iptables on start
```
systemctl enable iptables
systemctl enable ip6tables
```
### 1-12-6 Install csf dependencies
```
yum install wget perl unzip net-tools perl-libwww-perl perl-LWP-Protocol-https perl-GDGraph
```
### 1-12-7 Download and install csf
```
cd /opt
wget https://download.configserver.com/csf.tgz
tar -xzf csf.tgz
cd csf
sh install.sh
```
### 1-12-8 Remove installation files
```
rm -rf /opt/csf
rm /opt/csf.tgz
perl /usr/local/csf/bin/csftest.pl
```
### 1-12-9 Config csf
```
nano /etc/csf/csf.conf
```
Disable test mode and add port 443:
```
TESTING = "0"
TCP_IN = "21,22,25,80,110,143,443,465,587,636,990,993,995"
TCP_OUT = "21,22,25,80,110,143,443,465,587,636,990,993,995"
UDP_IN = "20,21,53,443"
UDP_OUT = "20,21,53,443"
```
### 1-12-10 Restart and test csf
```
systemctl restart {csf,lfd}
systemctl enable {csf,lfd}
systemctl is-active {csf,lfd}
csf -v
```
## 1-13 Config firewall and routing
```
nano \etc\csf\csfpre.sh
```
Write the following commands:
```
#!/bin/bash
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s 192.168.1.0/24 -j ACCEPT
iptables -A FORWARD -j REJECT
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```
## 1-14 Reload the csf
```
csf -r
```
## 1-15 Enable ip forwarding
```
nano /etc/sysctl.conf
```
Set the following config:
```
net.ipv4.ip_forward = 1
```
Check the config:
```
sysctl -p
```
## 1-16 Create username and password for VPN
```
touch /etc/ocserv/passwd
ocpasswd -c /etc/ocserv/passwd -g default username
```
Set the password for username
## 1-17 Start ocserv
```
systemctl start ocserv
systemctl enable ocserv
```
## 1-18 Check the ocserv
```
systemctl status ocserv
```
At this point ocserv must be active and you can connect to the server B using OpenConnect client apps.

# 2- Installing ocserv on server A
At first we need to install the ocserv on the server A similarly to server B.
## 2-1 Install epel repository:
```
yum install epel-release
```
## 2-2 Update Centos
```
yum update
```
## 2-3 Install ocserv
```
yum install ocserv gnutls-utils
```
## 2-4 Create cert directory
```
mkdir /etc/ocserv/cert
cd /etc/ocserv/cert/
```
## 2-5 Create a template file
```
touch ca.tmpl
yum install nano
nano ca.tmpl
```
Write these contents in the file:
```
cn = "VPN CA"
organization = "Big Corp"
serial = 1
expiration_days = -1
ca
signing_key
cert_signing_key
crl_signing_key
```
## 2-6 Generate certificates
```
certtool --generate-privkey --outfile ca-key.pem
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```
## 2-7 Create a server template
```
nano /etc/ocserv/server.tmpl
```
Write these contents in the template:
```
cn = "My server"
dns_name = "www.example.com"
organization = "MyCompany"
expiration_days = -1
signing_key
encryption_key
tls_www_server
```
## 2-8 Issue private key and certificate
```
certtool --generate-privkey --outfile server-key.pem
certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template /etc/ocserv/server.tmpl --outfile server-cert.pem
```
## 2-9 Create the ssl directory
```
mkdir /etc/ocserv/ssl/
```
## 2-10 Copy the certificate files to the ssl directory
```
cp ca-cert.pem server-key.pem server-cert.pem /etc/ocserv/ssl/
```
## 2-11 Config the ocserv
```
nano /etc/ocserv/ocserv.conf
```
find and edit the following settings:
```
auth = "plain[passwd=/etc/ocserv/passwd]"
server-cert = /etc/ocserv/ssl/server-cert.pem
server-key = /etc/ocserv/ssl/server-key.pem
ca-cert = /etc/ocserv/ssl/ca-cert.pem
ipv4-network = 192.168.1.0/24
```
Add the following lines to the file:
```
dns = 8.8.8.8
dns = 4.2.2.4
```
## 2-12 Install CSF
### 2-12-1 Disable firewalld
```
systemctl disable firewalld
systemctl stop firewalld
```
### 2-12-2 Install iptable-services
```
yum install iptables-services
```
### 2-12-3 Create config files for iptables
```
touch /etc/sysconfig/iptables
touch /etc/sysconfig/ip6tables
```
### 2-12-4 Start iptables
```
systemctl start iptables
systemctl start ip6tables
```
### 2-12-5 Enable iptables on start
```
systemctl enable iptables
systemctl enable ip6tables
```
### 2-12-6 Install csf dependencies
```
yum install wget perl unzip net-tools perl-libwww-perl perl-LWP-Protocol-https perl-GDGraph
```
### 2-12-7 Download and install csf
```
cd /opt
wget https://download.configserver.com/csf.tgz
tar -xzf csf.tgz
cd csf
sh install.sh
```
### 2-12-8 Remove installation files
```
rm -rf /opt/csf
rm /opt/csf.tgz
perl /usr/local/csf/bin/csftest.pl
```
### 2-12-9 Config csf
```
nano /etc/csf/csf.conf
```
Disable test mode and add port 443:
```
TESTING = "0"
TCP_IN = "21,22,25,80,110,143,443,465,587,636,990,993,995"
TCP_OUT = "21,22,25,80,110,143,443,465,587,636,990,993,995"
UDP_IN = "20,21,53,443"
UDP_OUT = "20,21,53,443"
```
### 2-12-10 Restart and test csf
```
systemctl restart {csf,lfd}
systemctl enable {csf,lfd}
systemctl is-active {csf,lfd}
csf -v
```
## 2-13 Config firewall and routing (This Step is different from what we did for server A)
```
nano \etc\csf\csfpre.sh
```
Write the following commands:
```
#!/bin/bash
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -j ACCEPT
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```
## 2-14 Reload the csf
```
csf -r
```
## 2-15 Enable ip forwarding
```
nano /etc/sysctl.conf
```
Set the following config:
```
net.ipv4.ip_forward = 1
```
Check the config:
```
sysctl -p
```
## 2-16 Create username and password for VPN
```
touch /etc/ocserv/passwd
ocpasswd -c /etc/ocserv/passwd -g default username
```
Set the password for username
## 2-17 Start ocserv
```
systemctl start ocserv
systemctl enable ocserv
```
## 2-18 Check the ocserv
```
systemctl status ocserv
```
At this point ocserv must be active and you can connect to the server A using OpenConnect client apps.
## 2-19 Route your ip to the ethernet
This is required because after connecting to the VPN server B the ip and default connection of the current server will be changed and you can not stay connected to the server anymore. Using the following command you can keep your ip (3.0.0.0) connected:
```
ip route add 3.0.0.0/32 via 1.0.0.1
```
## 2-20 Install jq
```
yum install jq
```
## 2-21 Route all iranian ips through the ethernet
```
cd ~
wget https://raw.githubusercontent.com/shahind/iran_ip_ranges/master/iran_ip_range.json
for range in $(jq .[] iran_ip_range.json | sed 's/"//g' | xargs); do
  ip route add $range via 1.0.0.1;
done;
```
For other countries, please refer to [this repository](https://github.com/shahind/iran_ip_ranges)
## 2-22 Install the openconnect client app
```
yum install openconnect
```
## 2-23 Connect to the server B
```
openconnect 2.0.0.0
```
Enter the username and password which you created in step 1-16. If you are using a ssh connection I suggest running in detached mode or using `tmux` as it keeps running even after you close your ssh connection to the server A. You can also wirte a simple code or use cronjob (`crontab` for example) to make sure that connection to the server B remains stable.
## 2-24 Done!
Everything must be fine at this level, your clients will reach Iranian websites through server A and foreign websites using server B. You need only one VPN user on server B and you must create VPN users server A as much as you need. Using a similar method you can chain also the server B to another server like C.

# 3 Troubleshouting
## 3-2 Only Irannian websites are accessible
Make sure your connection to server B from server A using `openconnect` is stablished. If you are using a ssh connection for reaching the server A, please keep it in mind that after closing your ssh connection the openconnect will become terminated as well. Please refer to section 2-23 for tips on how to keep `openconnect` connected.
## 3-1 Routings are missed
After a reboot, all routings we did in steps 2-19 to 2-21 may be reset. First check them by running ``ip route``, you must see a massive list of routings, if there are few default routings, you may need to re-do the steps 2-19 to 2-21 again and then conect to server B.

