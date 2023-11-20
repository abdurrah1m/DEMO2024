# Настройка статическго ip
## HQ-SRV
Смотрим название адаптера
```
ip a
```
`ens18`  
ip  
```
echo 192.168.0.40/24 > /etc/net/ifaces/ens18/ipv4address
```
gateway  
```
echo default via 192.168.0.1 > /etc/net/ifaces/ens18/ipv4route
```
Параметр BOOTPROTO отвечает за способ получения сетевой картой адреса, `static`  
```
nano /etc/net/ifaces/ens18/options
```
```
BOOTPROTO=static
```
Перезагрузка:
```
service network restart
```
или
```
systemctl restart network.service
```

NAT ISP:
```
iptables -A POSTROUTING -t nat -j MASQUERADE
```
```
iptables-save
```
```
nano /etc/net/scripts/nat
```
```
#!/bin/sh
/sbin/iptables -A POSTROUTING -t nat -j MASQUERADE
```
```
chmod +x /etc/net/scripts/nat
```
