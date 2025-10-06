- ISP
```tcl
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
systemctl restart network --now
apt-get update && apt-get install iptables -y
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable –now iptables
apt-get reinstall tzdata
timedatectl set-timezone Asia/Yekaterinburg
exec bash
```
