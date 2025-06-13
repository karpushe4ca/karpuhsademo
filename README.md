# Demo2025
## Изменение наименований машин
hostnamectl set-hostname isp.au-team.irpo;

## Создание связей между машинами\
bash
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
Нажать правой кнопкой мышки по сети  и настроить ipv4
IP - 192.168.10.2



