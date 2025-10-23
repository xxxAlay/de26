- ISP
```tcl
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p11] http://192.168.0.222/mirror p11/branch/x86_64 classic
rpm [p11] http://192.168.0.222/mirror p11/branch/noarch classic
rpm [p11] http://192.168.0.222/mirror p11/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/{ens20,ens21,ens22}
echo BOOTPROTO=static > /etc/net/ifaces/ens20/options
echo DISABLED=no >> /etc/net/ifaces/ens20/options
echo CONFIG_IPv4=yes >> /etc/net/ifaces/ens20/options
echo TYPE=eth >> /etc/net/ifaces/ens20/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens22/
echo BOOTPROTO=dhcp > /etc/net/ifaces/ens20/options
echo TYPE=eth >> /etc/net/ifaces/ens20/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
sed -i '10s/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
systemctl restart network --now
apt-get update && apt-get install iptables -y
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
apt-get reinstall tzdata
timedatectl set-timezone Asia/Yekaterinburg
systemctl enable iptables
exec bash

```

- HQ-RTR
```tcl
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
ntp timezon utc+5
username net_admin
p P@ssw0rd
r a
exit
int int0
ip add 172.16.1.4/28
ip n o
exit
port te0
ser te0/int0
encap un
exit
exit
int int0
connect port te0 service-instance te0/int0
exit
int int1
ip add 192.168.1.1/27
ip n i
exit
int int2
ip add 192.168.2.1/28
ip n i
exit
int int3
ip add 192.168.99.1/29
exit
port te1 
ser te1/int1
encap dot 100
rew pop 1
exit
ser te1/int2
encap dot 200
rew pop 1
exit
ser te1/int3
encap dot 999
rew pop 1
exit
exit
int int1
connect port te1 service-instance te1/int1
exit
int int2
connect port te1 service-instance te1/int2
exit
int int3
connect port te1 service-instance te1/int3
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
int tunnel.0
ip add 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.4 172.16.2.5 mode gre
ip ospf authentication-key ecorouter
exit
router ospf 1
net 172.16.0.0/30 a 0
net 192.168.1.0/27 a 0
net 192.168.2.0/28 a 0
passive-int default
no passive-int tunnel.0
area 0 authentication
exit
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload int int0
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
pool cli_pool 1
mask 255.255.255.240
gateway 192.168.2.1
dns 192.168.1.10
domain-name au-team.irpo
exit
exit
int int2
dhcp-server 1
exit
write memory

```

- BR-RTR
```tcl
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
ntp timezon utc+5
username net_admin
p P@ssw0rd
r a
exit
int int0
ip add 172.16.2.5/28
ip n o
exit
port te0
ser te0/int0
encap un
exit
exit
int int0
connect port te0 service-instance te0/int0
exit
int int1
ip add 192.168.3.1/28
ip n i
exit
port te1
ser te1/int1
encap un
exit
exit
int int1
connect port te1 service-instance te1/int1
exit
ip route 0.0.0.0 0.0.0.0 172.16.2.1
int tunnel.0
ip add 172.16.0.2/30
ip mtu 1400
ip tunnel 172.16.2.5 172.16.1.4 mode gre
ip ospf authentication-key ecorouter
exit
router ospf 1
net 172.16.0.0/30 a 0
net 192.168.3.0/28 a 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
ip nat pool NAT_POOL 192.168.3.1-192.168.3.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
write memory

```

- BR-SRV
```tcl
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.222/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname br-srv.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPv4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.3.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.3.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
useradd sshuser -u 2026
echo -e "P@ssw0rd\nP@ssw0rd" | passwd sshuser
sed -i '100s/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
echo -e "Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes" >> /etc/openssh/sshd_config
systemctl restart sshd
exec bash

```

- HQ-SRV
```tcl
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.222/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname hq-srv.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPv4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.1.10/27 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
apt-get update && apt-get install dnsmasq -y
systemctl enable --now dnsmasq
echo -e "no-resolv\ndomain=au-team.irpo\nserver=8.8.8.8\ninterface=*\naddress=/hq-rtr.au-team.irpo/192.168.1.1\nptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo\naddress=/br-rtr.au-team.irpo/192.168.3.1\naddress=/hq-srv.au-team.irpo/192.168.1.10\nptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo\naddress=/hq-cli.au-team.irpo/192.168.2.10\nptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo\naddress=/br-srv.au-team.irpo/192.168.3.10\naddress=/docker.au-team.irpo/172.16.1.1\naddress=/web.au-team.irpo/172.16.2.1" >> /etc/dnsmasq.conf
echo 192.168.1.1   hq-rtr.au-team.irpo >> /etc/hosts
systemctl restart dnsmasq
useradd sshuser -u 2026
echo -e "P@ssw0rd\nP@ssw0rd" | passwd sshuser
sed -i '100s/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
echo -e "Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes" >> /etc/openssh/sshd_config
systemctl restart sshd
exec bash
```

- HQ-CLI
```tcl
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.222/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64-i586 classic
EOF
hostnamectl set-hostname hq-cli.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "TYPE=eth\nBOOTPROTO=dhcp" > /etc/net/ifaces/ens20/options
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
exec bash

```
