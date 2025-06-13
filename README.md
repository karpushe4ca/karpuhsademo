# Demo2025
## 1. Изменение наименований машин
```bash
hostnamectl set-hostname isp.au-team.irpo;
exec bash
```
## Создание связей между машинами\
### ISP (nmtui)
    isp-hq  172.16.4.1/28
    isp-br  172.16.5.1/28
### HQ-RTR (nmtui)
    isp - hq-rtr - 172.16.4.2/28 gateway 172.16.4.1 
    hq - in  192.168.10.1/26
### BR-RTR (nmtui)
    isp - br-rtr - 172.16.5.2/28 gateway 172.16.5.1 
    br - in 192.168.60.1/27
### HQ-SRV (Graphics)
    IP - 192.168.10.2  Маска: 26  Шлюз: 192.168.10.1
### HQ-CLI (Graphics)
    IP - 192.168.10.3  Маска: 28  Шлюз: 192.168.10.1
### BR-SRV (Graphics)
    IP - 192.168.60.2  Маска: 27  Шлюз: 192.168.60.1
### BR-DC (Graphics)
    IP - 192.168.60.3  Маска: 27 (255.255.255.224)  Шлюз: 192.168.60.1
## Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV
    > Создание пользователей
    useradd -m -u 1010 sshuser
    passwd sshuser
    nano /etc/sudoers
    sshuser ALL=(ALL:ALL)NOPASSWD:ALL
## Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV
    nano /etc/mybanner
    Authorized access only
    nano /etc/openssh/sshd_config
    port 2024
    Banner /etc/mybanner
    MaxAuthTries 2
    AllowUsers sshuser
    systemctl restart sshd.service
## Сконфигурировать ip туннель HQ и BR
### ISP
    nano /etc/net/sysctl.conf
    net ipv4 forwarding = 1
### BR-RTR
    Название tun1
    mode: GRE
    Local: 172.16.5.2
    Remote: 172.16.4.2
    Address: 192.168.2.2/24
    Gateway: 192.168.2.1
### HQ-RTR
    Название tun1
    mode: GRE
    Local: 172.16.4.2
    Remote: 172.16.5.2
    Address: 192.168.2.1/24
    Gateway: 192.168.2.2
### Настройка протокола динамической конфигурации хостов HQ-RTR
    nano /etc/sysconfig/dhcpd
    DHCPARGS=ens35
    


    
