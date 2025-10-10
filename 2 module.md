# Samba
-- BR-SRV
```tsh
```

-- HQ-CLI
```tsg
```

# Raid

-- HQ-SRV
```tsg
apt-get update
apt-get install mdadm -y
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail -scan --verbose > /etc/mdadm.conf
apt-get install fdisk -y
echo -e "n\np\n1\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo "/dev/md0p1 /raid ext4 defaults 0 0" >> /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.10(rw,sync,no_subtree_check)" >> /etc/exports
systemctl enable --now nfs-server
exportfs -a
```

-- HQ-CLI
```
apt-get update
apt-get install nfs-common -y
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs intr,soft,_netdev,x-systemd.automount 0 0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test
```

# Chrony

-- ISP
```
apt-get install –y chrony
	rm -rf /etc/chrony.conf
echo -e "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow 0/0" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
chronyc tracking | grep Stratum

```

-- HQ-RTR
```
en
	conf t
	ntp server 172.16.1.1
	ntp timezone utc+5
	exit
	write memory

```


-- BR-RTR
```
en
	conf t
	ntp server 172.16.2.1
	ntp timezone utc+5
	exit
	write memory

```

-- HQ-CLI
```
apt-get install -y chrony
echo server 172.16.1.1 iburst prefer >> /etc/chrony.conf	
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl

```

-- HQ-SRV
```
apt-get install -y chrony
echo server 172.16.1.1 iburst prefer >> /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl

```

-- BR-SRV
```
apt-get install -y chrony
echo server 172.16.2.1 iburst prefer > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl


--
```
apt-get update
apt-get install ansible -y
	cat <<EOF > /etc/ansible/hosts
Hosts:
 hosts:
  HQ-SRV:
    ansible_host: 192.168.1.10
    ansible_user: deviate_user
    ansible_port: 2026
  HQ-CLI:
    ansible_host: 192.168.2.10
    ansible_user: remote_user
    ansible_port: 2026
  HQ-RTR:
    ansible_host: 192.168.1.1
    ansible_user: net.admin
    ansible_password: Passw0rd
    ansible_connection: network_cli
    ansible_network_os: ios
  BR-RTR:
    ansible_host: 192.168.3.1
    ansible_user: net.admin
    ansible_password: Passw0rd
    ansible_connection: network_cli
    ansible_network_os: ios
