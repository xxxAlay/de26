#Samba
-- BR-SRV
```tsh
```

-- HQ-CLI
```tsg
```

#Raid
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
