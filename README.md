# Demo2025
## 1. Изменение наименований машин
```bash
hostnamectl set-hostname isp.au-team.irpo;
exec bash
```
## 2. Создание связей между машинами
### ISP (nmtui)
```bash
isp-hq  172.16.4.1/28
isp-br  172.16.5.1/28
```
### HQ-RTR (nmtui)
```bash
isp - hq-rtr - 172.16.4.2/28 gateway 172.16.4.1 
hq - in  192.168.10.1/26
```
### BR-RTR (nmtui)
```bash
isp - br-rtr - 172.16.5.2/28 gateway 172.16.5.1 
br - in 192.168.60.1/27
```
### HQ-SRV (Graphics)
```bash
IP - 192.168.10.2  Маска: 26  Шлюз: 192.168.10.1
```
### HQ-CLI (Graphics)
```bash
IP - 192.168.10.3  Маска: 28  Шлюз: 192.168.10.1
```
### BR-SRV (Graphics)
```bash
IP - 192.168.60.2  Маска: 27  Шлюз: 192.168.60.1
```
### BR-DC (Graphics)
```bash
IP - 192.168.60.3  Маска: 27 (255.255.255.224)  Шлюз: 192.168.60.1
```
## 3. Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV
```bash
Создание пользователей
useradd -m -u 1010 sshuser
passwd sshuser
# P@ssw0rd
```
```
nano /etc/sudoers
```
```
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
```
## 3.1 Создайте пользователя net_admin на серверах HQ-SRV и BR-SRV
```bash
adduser net_admin
```
```
passwd net_admin
# P@$$word
```
```
nano /etc/sudoers
```
```
net_admin ALL=(ALL:ALL)NOPASSWD:ALL
```
## 4. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV
```bash
nano /etc/mybanner
```
```
Authorized access only
```
```
nano /etc/openssh/sshd_config
```
```
port 2024
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser
```
```
systemctl restart sshd.service
```
## 5. Сконфигурировать ip туннель HQ и BR
### ISP
```bash
nano /etc/net/sysctl.conf
```
```
net ipv4 forwarding = 1
```
### BR-RTR
```bash
Название tun1
mode: GRE
Local: 172.16.5.2
Remote: 172.16.4.2
Address: 192.168.0.2/24
Gateway: 192.168.0.1
```
### HQ-RTR
```bash
Название tun1
mode: GRE
Local: 172.16.4.2
Remote: 172.16.5.2
Address: 192.168.0.1/24
Gateway: 192.168.0.2
# Перезагрузить BR-RTR и HQ-RTR и не забыть перезапустить интернет и ISP
```
### 6. Настройка протокола динамической конфигурации хостов HQ-RTR
```bash
nano /etc/sysconfig/dhcpd
```
```
DHCPARGS=ens35
```
```
cp /etc/dhcp/dhcpd.conf{.example,}
```
```
nano /etc/dhcp/dhcpd.conf
```
```
option domain-name "au-team.irpo";
option domain-name-servers 192.168.10.2;

default-lease-time 6000;
max-lease-time 72000;

authoritative;
subnet 192.168.10.0 netmask 255.255.255.192 {
    range 192.168.10.10 192.168.10.20;
    option routers 192.168.10.1;
}

```
```
systemctl enable –-now dhcpd
```
### 7. Настройка OSPF HQ-RTR И BR-RTR
```bash
#Отключить FireWall systemctl disable firewalld --now hq-r
```
```
nano /etc/frr/daemons
```
```
ospfd=yes
```
### HQ-RTR
```bash
systemctl enable --now frr
```
```
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 192.168.10.0/26 area 0 
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
```
```
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
```
### BR-RTR
```bash
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 192.168.60.0/27 area 0 
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
```
```
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
# Перезагрузить BR-RTR и HQ-RTR
```
### Проверка работы OSPF
```bash
vtysh
```
```
show ip ospf neighbor
```
### 8. Настройка chrony HQ-SRV
```bash
/etc/chrony.conf
```
```
server 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0
```
```
systemctl enable --now chronyd
```
```
chronyc sources
```
```
chronyc tracking | grep Stratum (Должно равняться 5)
```
### 9. Создание RAID 5 на HQ-SRV
```bash
#Создаем диски, после создания:
lsblk (Выводит диски)
```
```
# Обнуляем суперблоки 
mdadm --zero-superblock --force /dev/sd{b,c,d}
Вывод: mdadm: Unrecongised md component device - /dev/sdx
```
```
# Удаляем старые данные
wipefs --all --force /dev/sd{b,c,d}
```
### Создание RAID
```bash
mdadm --create /dev/md0 -l 5 -n 3 /dev/sd{b,c,d}
```
```
# Проверяем, должен быть в конце raid5
lsblk
```
```
# Создаем фалйовую систему
mkfs -t ext4 /dev/md0
```
```
mkdir /etc/mdadm
```
```
#Заполняем файл
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```
```
#Создаем директорию
mkdir /mnt/raid5
```
```
/etc/fstab
```
```
/dev/md0  /mnt/raid5  ext4  defaults  0  0
```
```
mount -a
```
```
df -h (Проверяем)
```
### 10. Настройте принт-сервер cups на сервере HQ-SRV. (Если будет работать)
### HQ-SRV   
```bash
usermod -aG lpadmin
```
```
systemctl enable cups
systemctl start cups
```
```
nano /etc/cups/cupsd.conf
```
```
Port 631

<Location />
  Order allow,deny
  Allow all
</Location>

<Location /admin>
  Order allow,deny
  Allow all
</Location>

```
```
sudo systemctl restart cups
```
```
# Если он не создан, ты можешь создать его вручную через веб-интерфейс или командой
lpadmin -p PDF -E -v cups-pdf:/ -m drv:///cupsfilters/PDF.ppd
```
```
lpadmin -p PDF -o printer-is-shared=true
```
```
cupsctl --share-printers
```
### HQ-CLI
```bash
nano /etc/cups/client.conf
```
```
ServerName 192.168.10.2
```
```
sudo systemctl restart cups
```
```
# Убедиться что работает принтер
lpstat -a 
```
```
lpoptions -d PDF
```
```
# Создание тестового PDF файла
echo "Test page from $(hostname)" | enscript -o - | ps2pdf - test.pdf
(echo "Test page from $(hostname)" > test.pdf) если не сработает верхний
```
```
lp -d PDF test.pdf
```
### 11. Настройка DNS для офисов HQ-SRV и BR-SRV (Если успею!)
### HQ-SRV
```bash
nano /etc/bind/options.conf
```
```
listen on { any; };
```
```
forward first;
forwarders { 77.88.8.8; };
```
```
systemctl enable bind —now
```
```
nano /etc/bind/local.conf
```
```
zone "au-team.irpо" {
        type master;
        file "au.db";
};
```
```
zone "8.16.172. in-addr.arpa" {
        type master;
        file "0.db";
};
```
```
cd /etc/bind/zone
```
```
cp localdomain au.db
```
```
cp 127.in-addr.arpa 0.db
```
```
Chown root:named {au,0}.db
```
```
nano au.db
```
```
hq-rtr    192.168.10.1
hq-srv    192.168.10.2
hq-cli    192.168.10.3
br-rtr    192.168.60.1
br-srv    192.168.60.2
au-team.irpo. root.au-team.irpo.
moodle hq-rtr.au-team.irpo.
wiki hq-rtr.au-team.irpo.
```
```
nano 0.db
```
```
au-team.irpo. root.au-team.irpo.
1 IN PTR hq-rtr.au-team.irpo.
2 IN PTR hq-srv.au-team.irpo.
3 IN PTR hq-cli.au-team.irpo.
```
```
Systemctl restart bind
```
### Поднять AD DS в WinSrv









