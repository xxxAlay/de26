# Samba
-- HQ-SRV
```tsg
echo "server=/au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
systemctl restart dnsmasq
```

-- BR-SRV
```tsh
if ! grep -q '^nameserver 8\.8\.8\.8$' /etc/resolv.conf; then
    echo 'nameserver 8.8.8.8' >> /etc/resolv.conf
fi
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
if ! grep -q '^nameserver 192\.168\.1\.10$' /etc/resolv.conf; then
    echo nameserver 192.168.1.10 >> /etc/resolv.conf
fi
sleep 2
echo 192.168.3.10 br-srv.au-team.irpo >> /etc/hosts
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc --option='dns forwarder=192.168.1.10'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba.service
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif
```

-- HQ-CLI
```tsg
systemctl restart network
apt-get update && apt-get install bind-utils -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
```
```tsg
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
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
```

# Ansible

-- BR-SRV
```
apt-get update
apt-get install ansible -y
	cat <<EOF > /etc/ansible/hosts
Hosts:
 hosts:
  HQ-SRV:
    ansible_host: 192.168.1.10
    ansible_user: ssh_user
    ansible_port: 2026
  HQ-CLI:
    ansible_host: 192.168.2.10
    ansible_user: ssh_user
    ansible_port: 2026
  HQ-RTR:
    ansible_host: 192.168.1.1
    ansible_user: net_admin
    ansible_password: P@ssw0rd
    ansible_connection: network_cli
    ansible_network_os: ios
  BR-RTR:
    ansible_host: 192.168.3.1
    ansible_user: net_admin
    ansible_password: P@ssw0rd
    ansible_connection: network_cli
    ansible_network_os: ios
EOF
sed -i '11a\
ansible_python_interpreter=/usr/bin/python3\
interpreter_python=auto_silent\
ansible_host_key_checking=false' /etc/ansible/ansible.cfg

ssh-keygen -t rsa
```

-- HQ-CLI
```
useradd ssh_user -u 2026
echo -e "P@ssw0rd\nP@ssw0rd" | passwd ssh_user
sed -i '100s/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "ssh_user" wheel
echo -e "Port 2026\nAllowUsers ssh_user\nMaxAuthTries 2\nPasswordAuthentication yes" >> /etc/openssh/sshd_config
systemctl restart sshd

```

--BR-SRV
```
ssh-copy-id -p 2026 ssh_user@192.168.1.10
ssh-copy-id -p 2026 ssh_user@192.168.2.10
ansible all -m ping

```


# Docker
-- BR-SRV
```tcl
apt-get update && apt-get install docker-compose docker-engine -y
systemctl enable --now docker
mkdir /mnt/addmount
mount /dev/sr0 /mnt/addmount -o ro
mount -o loop /dev/sr0
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
cat > docker-compose.yml << 'EOF'
services:
  db:
    image: mariadb
    container_name: db
    environment:
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      MYSQL_ROOT_PASSWORD: Passw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: Passw0rd
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app_network
    restart: unless-stopped
  testapp:
    image: site
    container_name: testapp
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      DB_PORT: 3306
    ports:
      - "8080:8000"
    networks:
      - app_network
    depends_on:
      - db
    restart: unless-stopped
volumes:
  db_data:
networks:
  app_network:
    driver: bridge
EOF
docker compose -f site.yml up -d
docker compose up -d && sleep 5 && docker exec -it db mysql -u root -p'Passw0rd' -e "CREATE DATABASE IF NOT EXISTS testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"

```

# Web
```
apt-get update
apt-get install -y apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-{opcache,curl,gd,intl,mysqli,xml,xmlrpc,ldap,zip,soap,mbstring,json,xmlreader,fileinfo,sodium}
apt-get install expect -y
mount -o loop /dev/sr0
systemctl enable --now httpd2 mysqld
expect << 'EOF'
spawn mysql_secure_installation
sleep 1
send "\r"          ;# current password – just Enter
sleep 1
send "n\r"         ;# unix_socket authentication
sleep 1
send "Y\r"         ;# change root password
sleep 1
send "P@ssw0rd\r"  ;# new password
sleep 1
send "P@ssw0rd\r"  ;# confirm password
sleep 1
send "Y\r"         ;# remove anonymous users
sleep 1
send "Y\r"         ;# disallow root login remotely
sleep 1
send "Y\r"         ;# remove test database
sleep 1
send "Y\r"         ;# reload privileges
sleep 1
expect eof
EOF
sleep 2
mariadb -u root -pP@ssw0rd -e "
CREATE DATABASE webdb;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';
FLUSH PRIVILEGES;
"
mkdir /tmp/tmpadd
cp -rf /mnt/additional /tmp/tmpadd
cp -rf /tmp/tmpadd/additional/web/dump.sql /tmp/tmpadd/additional/web/dump.sql.bak
iconv -f UTF-16LE -t UTF-8 /tmp/tmpadd/additional/web/dump.sql -o /tmp/tmpadd/additional/web/dump_utf8.sql
mariadb -u root -p 'P@ssw0rd' webdb < /tmp/dump_utf8.sql
mysql -u root -p'P@ssw0rd' webdb -e "SHOW TABLES;"
chown apache:apache /var/www/html
chown apache:apache /var/www/webdata
cp /tmp/tmpadd/additional/web/index.php /var/www/html/index.php
cd /var/www/html
cp -rf /media/ALTLinux/web/* /var/www/html
rm -rf /var/www/html/index.html
sed -i 's/\$username = "user";/\$username = "webc";/' /var/www/html/index.php
sed -i 's/\$password = "password";/\$password = "P@ssw0rd";/' /var/www/html/index.php
sed -i 's/\$dbname = "db";/\$dbname = "webdb";/' /var/www/html/index.php
systemctl enable --now httpd2
systemctl restart httpd2
```
