# Конфигурация сетевой инфраструктуры

Данный документ содержит инструкции по настройке сетевой инфраструктуры, включающей ISP, маршрутизаторы HQ-RTR и BR-RTR, а также серверы HQ-SRV и BR-SRV.

---

## ISP

### Настройка hostname
```bash
hostnamectl set-hostname ISP; exec bash
```

### Настройка сетевых интерфейсов

**Интерфейс ens19:**
```bash
mkdir /etc/net/ifaces/ens19
echo "172.16.1.1/28" > /etc/net/ifaces/ens19/ipv4address
echo BOOTPROTO=static > /etc/net/ifaces/ens19/options
echo TYPE=eth >> /etc/net/ifaces/ens19/options
```

**Интерфейс ens20:**
```bash
mkdir /etc/net/ifaces/ens20
echo 172.16.2.1/28 > /etc/net/ifaces/ens20/ipv4address
echo BOOTPROTO=static > /etc/net/ifaces/ens20/options
echo TYPE=eth >> /etc/net/ifaces/ens20/options
```

### Включение IP forwarding
Отредактировать файл `/etc/net/sysctl.conf` (поменять forward на 1)

### Перезапуск сети
```bash
systemctl restart network
```

### Настройка NAT и iptables
```bash
apt-get update
apt-get install iptables
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save -f /etc/sysconfig/iptables
systemctl enable --now iptables
```

### Настройка часового пояса
```bash
apt-get install tzdata -y
timedatectl set-timezone Asia/Yekaterinburg
```

---

## HQ-RTR

### Базовая конфигурация
```
en
conf t
hostname hq-rtr.net16tech.institute
```

### Настройка интерфейсов

**Интерфейс ISP:**
```
int isp
ip add 172.16.1.2/28
ip nat outside
```

**Интерфейс HQ.100:**
```
int hq.100
ip add 192.168.0.1/26
ip nat inside
```

**Интерфейс HQ.200:**
```
int hq.200
ip add 192.168.1.1/27
ip nat inside
```

**Интерфейс HQ.999:**
```
int hq.999
ip add 192.168.2.1/28
ip nat inside
```

### Настройка портов и VLAN

**Порт te0 (ISP):**
```
port te0
service-instance isp
encapsulation untagged
connect ip int isp
ex
```

**Порт te1 (HQ VLANs):**
```
port te1
service-instance hq.100
encapsulation dot1q 100
rewrite pop 1
con ip int hq.100
ex
service-instance hq.200
encapsulation dot1q 200
rewrite pop 1
con ip int hq.200
ex
service-instance hq.999
encapsulation dot1q 999
rewrite pop 1
con ip int hq.999
ex
ex
```

### Настройка GRE туннеля
```
int tunnel.0
ip mtu 1400
ip tunnel 172.16.1.2 172.16.2.2 mode gre
ip address 10.0.0.1/30
ex
```

### Настройка OSPF
```
router ospf 1
router-id 1.1.1.1
passive-interface default
no passive-interface tunnel.0
network 192.168.0.0/26 area 0
network 192.168.1.0/27 area 0
network 192.168.2.0/28 area 0
network 10.0.0.0/30 area 0
exit
```

### Настройка OSPF аутентификации
```
int tunnel.0
ip ospf authentication-key cisco
ip ospf authentication
exit
```

### Настройка DHCP
```
ip pool DHCP 1
range 192.168.1.2-192.168.1.30
exit

dhcp-server 1
pool DHCP 1
mask 26
gateway 192.168.1.1
dns 192.168.0.2
domain-search net16tech.institute
domain-name net16tech.institute
ex
exit
```

### Настройка NAT
```
ip nat pool NAT 192.168.0.1-192.168.0.62,192.168.1.1-192.168.1.30,192.168.2.1-192.168.2.14
ip nat source dynamic inside-to-outside pool NAT overload int isp
```

### Маршрутизация и финальные настройки
```
ip route 0.0.0.0/0 172.16.1.1
int hq.200
dhcp-server 1
ntp timezone utc+5
write
```

---

## BR-RTR

### Базовая конфигурация
```
en
conf t
hostname br-rtr.net16tech.institute
```

### Настройка интерфейсов

**Интерфейс ISP:**
```
int isp
ip add 172.16.2.2/28
ip nat outside
```

**Интерфейс BR:**
```
int br
ip add 192.168.3.1/29
ip nat inside
```

### Настройка портов
```
port te0
service-instance isp
encapsulation untagged
con ip int isp

port te1
service-instance br
encapsulation untagged
con ip int br
ex
ex
```

### Настройка GRE туннеля
```
int tunnel.0
ip mtu 1400
ip tunnel 172.16.2.2 172.16.1.2 mode gre
ip address 10.0.0.2/30
ip ospf authentication-key cisco
ip ospf authentication
exit
```

### Настройка OSPF
```
router ospf 1
router-id 2.2.2.2
network 192.168.3.0/29 area 0
network 10.0.0.0/30 area 0
passive-interface default
no passive-interface tunnel.0
ex
```

### Настройка NAT и маршрутизации
```
ip nat pool NAT 192.168.3.1-192.168.3.6
ip nat source dynamic inside-to-outside pool NAT overload int isp
ip route 0.0.0.0/0 172.16.2.1
ntp timezone utc+5
write
```

---

## HQ-SRV

### Настройка hostname
```bash
hostnamectl hostname hq-srv.net16tech.institute; exec bash
```

### Настройка сети
```bash
echo 192.168.0.2/27 > /etc/net/ifaces/ens18/ipv4address
echo BOOTPROTO=static > /etc/net/ifaces/ens18/options
echo TYPE=eth >> /etc/net/ifaces/ens18/options
echo "default via 192.168.0.1" > /etc/net/ifaces/ens18/ipv4route
echo nameserver 8.8.8.8 > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
```

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Yekaterinburg
```

### Создание пользователя sshuser
```bash
useradd sshuser -u 2016
passwd sshuser
usermod -aG wheel sshuser
```

### Настройка sudo
Добавить строку в `/etc/sudoers`:
```
sshuser ALL=(ALL) NOPASSWD:ALL
```

### Настройка SSH
Привести указанные строки в файле `/etc/openssh/sshd_config` к следующим значениям:
```
Port 2216
MaxAuthTries 1
PasswordAuthentication yes
Banner /etc/openssh/bannermotd
AllowUsers  sshuser
```
> **Примечание:** В параметре AllowUsers вместо пробела используется Tab

Создать файл `/etc/openssh/bannermotd`:
```
----------------------
Authorized access only
----------------------
```

Перезапустить SSH:
```bash
systemctl restart sshd
```

### Установка и настройка BIND DNS

**Установка:**
```bash
apt-get update
apt-get install bind bind-utils -y
```

**Настройка `/etc/bind/options.conf`:**
```
options {
        version "unknown";
        directory "/etc/bind/zone";
        dump-file "/var/run/named/named_dump.db";
        statistics-file "/var/run/named/named.stats";
        recursing-file "/var/run/named/named.recursing";
        secroots-file "/var/run/named/named.secroots";

        pid-file none;

        listen-on { 192.168.0.2; };
        listen-on-v6 { none; };

        forwarders { 8.8.8.8; };

        allow-query { any; };
        allow-recursion { any; };
};

logging {
        // Logging configuration (commented out by default)
};
```

**Настройка зоны `/etc/bind/zone/net16tech.institute`:**
```
$TTL    1D
@       IN      SOA     net16tech.institute. root.net16tech.institute. (
                                2025110500      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      net16tech.institute.
        IN      A       192.168.0.2
HQ-RTR  IN      A       192.168.0.1
HQ-RTR  IN      A       192.168.1.1
HQ-SRV  IN      A       192.168.0.2

BR-RTR  IN      A       192.168.3.1
BR-SRV  IN      A       192.168.3.2
HQ-CLI  IN      A       192.168.1.2
docker  IN      A       172.16.1.1
web     IN      A       172.16.2.2
```

**Настройка обратной зоны `/etc/bind/zone/168.192.in-addr.arpa`:**
```
$TTL    1D
@       IN      SOA     net16tech.institute. root.net16tech.institute. (
                                2025110500      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      net16tech.institute.
0.1     IN      PTR     HQ-RTR.net16tech.institute
1.1     IN      PTR     HQ-RTR.net16tech.institute
0.2     IN      PTR     HQ-SRV.net16tech.institute
1.2     IN      PTR     HQ-CLI.net16tech.institute
```

**Настройка `/etc/bind/local.conf`:**
```
zone "net16tech.institute" {
type master;
file "net16tech.institute";
};

zone "168.192.in-addr.arpa" {
type master;
file "168.192.in-addr.arpa";
};
```

**Установка прав доступа:**
```bash
cd /etc/bind/zone
chmod 700 *
chown named. *
```

**Настройка DNS resolver:**
```bash
echo nameserver 192.168.0.2 > /etc/net/ifaces/ens18/resolv.conf
echo search net16tech.institute >> /etc/net/ifaces/ens18/resolv.conf
```

**Запуск BIND:**
```bash
systemctl enable --now bind
```

---

## BR-SRV

### Настройка hostname
```bash
hostnamectl hostname br-srv.net16tech.institute; exec bash
```

### Настройка сети
```bash
echo 192.168.3.2/29 > /etc/net/ifaces/ens18/ipv4address
echo BOOTPROTO=static > /etc/net/ifaces/ens18/options
echo TYPE=eth >> /etc/net/ifaces/ens18/options
echo "default via 192.168.3.1" > /etc/net/ifaces/ens18/ipv4route
echo search net16tech.institute >> /etc/net/ifaces/ens18/resolv.conf
echo nameserver 192.168.0.2 >> /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
```

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Yekaterinburg
```

### Создание пользователя sshuser
```bash
useradd sshuser -u 2016
passwd sshuser
usermod -aG wheel sshuser
```

### Настройка sudo
Добавить строку в `/etc/sudoers`:
```
sshuser ALL=(ALL) NOPASSWD:ALL
```

### Настройка SSH
Привести указанные строки в файле `/etc/openssh/sshd_config` к следующим значениям:
```
Port 2216
MaxAuthTries 1
PasswordAuthentication yes
Banner /etc/openssh/bannermotd
AllowUsers  sshuser
```
> **Примечание:** В параметре AllowUsers вместо пробела используется Tab

Создать файл `/etc/openssh/bannermotd`:
```
----------------------
Authorized access only
----------------------
```

Перезапустить SSH:
```bash
systemctl restart sshd
```

---

## Сводная информация

### Сетевая топология

| Устройство | Интерфейс | IP-адрес | Назначение |
|------------|-----------|----------|------------|
| ISP | ens19 | 172.16.1.1/28 | Подключение к HQ-RTR |
| ISP | ens20 | 172.16.2.1/28 | Подключение к BR-RTR |
| HQ-RTR | isp | 172.16.1.2/28 | Внешний интерфейс |
| HQ-RTR | hq.100 | 192.168.0.1/26 | Внутренняя сеть (серверы) |
| HQ-RTR | hq.200 | 192.168.1.1/27 | Внутренняя сеть (клиенты) |
| HQ-RTR | hq.999 | 192.168.2.1/28 | Управляющая сеть |
| HQ-RTR | tunnel.0 | 10.0.0.1/30 | GRE туннель |
| BR-RTR | isp | 172.16.2.2/28 | Внешний интерфейс |
| BR-RTR | br | 192.168.3.1/29 | Внутренняя сеть |
| BR-RTR | tunnel.0 | 10.0.0.2/30 | GRE туннель |
| HQ-SRV | ens18 | 192.168.0.2/27 | Сервер (DNS) |
| BR-SRV | ens18 | 192.168.3.2/29 | Сервер |

### Ключевые параметры

- **Часовой пояс:** Asia/Yekaterinburg (UTC+5)
- **SSH порт:** 2216
- **SSH пользователь:** sshuser (UID: 2016)
- **DNS сервер:** 192.168.0.2 (HQ-SRV)
- **Домен:** net16tech.institute
- **OSPF аутентификация:** cisco
- **GRE туннель:** между HQ-RTR и BR-RTR
