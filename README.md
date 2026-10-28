# Домашнее задание 1. Сетевой стек

## Задание 1. Анализ состояний TCP-соединений
### Шаг 1. Запуск сервера:
```
python3 -m http.server 8080
```
Проверим, что появился слушающий TCP сокет на порту 8080:
```
denich@denich-LOQ-15IRX9:~$ ss -lntp | grep 8080
LISTEN 0      5                 0.0.0.0:8080       0.0.0.0:*    users:(("python3",pid=20261,fd=3))  
```
### Шаг 2. Подключение через curl

Вывод отбросим, нам важен сам факт соединения
```
denich@denich-LOQ-15IRX9:~$ curl http://127.0.0.1:8080 > /dev/null
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2087  100  2087    0     0   695k      0 --:--:-- --:--:-- --:--:-- 1019k
```

### Шаг 3. Анализ состояния TCP-сокетов для порта 8080
После запроса проанализируем состояние TCP-сокетов для порта 8080:

```
denich@denich-LOQ-15IRX9:~$ ss -tanp | grep 8080
LISTEN    0      5                 0.0.0.0:8080          0.0.0.0:*     users:(("python3",pid=20261,fd=3))       
TIME-WAIT 0      0               127.0.0.1:8080        127.0.0.1:37838  
```
В строке `TIME-WAIT 0 0 127.0.0.1:8080 127.0.0.1:37838` локальный адрес сокета — 127.0.0.1:8080 (порт сервера), peer — 127.0.0.1:37838 (динамический порт клиента curl). Поскольку в состоянии TIME-WAIT остаётся сторона, инициировавшая закрытие, нахождение в TIME-WAIT серверного сокета означает, что соединение первым закрыл сервер. Это соответствует поведению python3 -m http.server: он работает по HTTP/1.0 без keep-alive и закрывает соединение после отправки ответа.

Роль TIME-WAIT
1) Надёжное завершение соединения. 
Пакет может задержаться в сети до MSL (maximum segment lifetime). Если финальный ACK потерялся, удалённая сторона может повторно отправить свой FIN. TIME_WAIT даёт достаточно времени, чтобы система успела получить этот повторный FIN и отправила к нему свой ACK, прежде чем соединение окончательно закроется. Если соединение закроется раньше, потерянный FIN может привести к тому, что удалённая сторона никогда не получит финальное ACK и соединение не будет корректно закрыто. 

2) Защита от запоздавших сегментов.
Пакет может задержаться в сети до MSL. Если 4-кортеж (локальный адрес:порт, удалённый:порт) переиспользовать сразу после закрытия, запоздавший сегмент старого соединения попадёт в новое соединение → порча данных или RST. TIME-WAIT держит 4-кортеж занятым 2·MSL, чтобы все старые пакеты гарантированно умерли в сети до переиспользования. 

Сокет в TIME-WAIT не удаляют принудительно, потому что он ещё нужен для корректной работы TCP. Ядро снимает его само по истечении 2·MSL времени.

### Шаг 4. Проблемы при большом количестве TIME-WAIT
- **Исчерпание ephemeral-портов.** Много коротких исходящих соединений
  без пула/keep-alive → порты не успевают освобождаться → новые
  `connect()` падают с `EADDRNOTAVAIL` / `Cannot assign requested address`.
- **Вторично:** рост памяти под сокеты, шум в диагностике `ss`.

## Задание 2. Динамическая маршрутизация с BIRD
### Шаг 1. Создание dummy-интерфейса service_0, service_1, service_2, srv_1

service_0
```
sudo ip link add name service_0 type dummy
sudo ip link set service_0 up
sudo ip addr add 192.168.14.88/32 dev service_0
```
service_1
```
sudo ip link add name service_1 type dummy
sudo ip link set service_1 up
sudo ip addr add 192.168.14.1/30 dev service_1
```
service_2
```
sudo ip link add name service_2 type dummy
sudo ip link set service_2 up
sudo ip addr add 192.168.10.4/32 dev service_2
```

srv_1
```
sudo ip link add name srv_1 type dummy
sudo ip link set srv_1 up
sudo ip addr add 192.168.14.4/32 dev srv_1
```

Проверка, что ядро видит созданные интерфейсы:
```
denich@denich-LOQ-15IRX9:~$ ip route show table local | grep -E 'service_|srv_'
local 192.168.10.4 dev service_2 proto kernel scope host src 192.168.10.4 
local 192.168.14.1 dev service_1 proto kernel scope host src 192.168.14.1 
broadcast 192.168.14.3 dev service_1 proto kernel scope link src 192.168.14.1 
local 192.168.14.4 dev srv_1 proto kernel scope host src 192.168.14.4 
local 192.168.14.88 dev service_0 proto kernel scope host src 192.168.14.88 
```

### Шаг 2. Настройка конфига BIRD

Вставим конфигурацию для задания:
```
denich@denich-LOQ-15IRX9:~$ sudo cat /etc/bird/bird.conf
router id 192.168.14.88;

protocol device {
    scan time 10;
}

protocol direct {
    ipv4;
    interface "service_*";
}

protocol kernel {
    ipv4 { export all; };
    scan time 20;
}

filter rip_export {
    if (net ~ 192.168.14.0/24 && net.len = 32) then accept;
    reject;
}

protocol rip {
    ipv4 {
        import all;
        export filter rip_export;
    };
    interface "enp7s0" {
        mode multicast;
    };
}
```
Проверка:

все компоненты маршрутизации активны
```
denich@denich-LOQ-15IRX9:~$ sudo birdc show protocols
BIRD 2.14 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     21:03:10.514  
direct1    Direct     ---        up     21:03:10.514  
kernel1    Kernel     master4    up     21:03:10.514  
rip1       RIP        master4    up     21:03:10.514
```

Проверка таблицы BIRD  
srv_1 отсутствует, так как protocol direct импортирует только service_*.
```
denich@denich-LOQ-15IRX9:~$ sudo birdc show route
BIRD 2.14 ready.
Table master4:
192.168.14.0/30      unicast [direct1 21:03:10.515] * (240)
	dev service_1
192.168.14.88/32     unicast [direct1 21:03:10.515] * (240)
	dev service_0
192.168.10.4/32      unicast [direct1 21:03:10.515] * (240)
	dev service_2
```

Проверка экспорта RIP  
Остальные адреса отсечены фильтром (/30 и чужая подсеть).
```
denich@denich-LOQ-15IRX9:~$ sudo birdc show route export rip1
BIRD 2.14 ready.
Table master4:
192.168.14.88/32     unicast [direct1 21:03:10.515] * (240)
	dev service_0
```
### Шаг 3. Доказательство tcpdump

```
denich@denich-LOQ-15IRX9:~$ sudo tcpdump -ni enp7s0 -v 'udp port 520'
tcpdump: listening on enp7s0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:23:47.876708 IP (tos 0xc0, ttl 1, id 44923, offset 0, flags [none], proto UDP (17), length 52)
    192.168.147.54.520 > 224.0.0.9.520: 
	RIPv2, Response, length: 24, routes: 1 or less
	  AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
```
В RIP-анонсе присутствует только 192.168.14.88/32 (routes: 1 or less). Лишних адресов нет. Это доказывает, что фильтр работает корректно: service_1 (/30), service_2 (другая подсеть) и srv_1 (имя не service_*) не анонсируются.

## Задание 3. Настройка фаервола/ Host Firewalling

### Шаг 1: Настройка iptables

```
denich@denich-LOQ-15IRX9:~$ sudo iptables -A INPUT -p tcp --dport 8080 -j DROP
```

Проверка, что правило добавлено:
```
denich@denich-LOQ-15IRX9:~$ sudo iptables -L INPUT -n --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    DROP       6    --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080
```

### Шаг 2: Запуск тестового веб-сервера

```
denich@denich-LOQ-15IRX9:~$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

### Шаг 3: Запуск перехвата трафика через tcpdump

```
denich@denich-LOQ-15IRX9:~$ sudo tcpdump -i any -n port 8080
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
```
### Шаг 4: Проверка работы фаервола

Попробуем при помощи curl подключиться к серверу:
```
denich@denich-LOQ-15IRX9:~$ curl --connect-timeout 3 http://127.0.0.1:8080
curl: (28) Failed to connect to 127.0.0.1 port 8080 after 3002 ms: Timeout was reached
```

Посмотрим на tcpdump:

```
denich@denich-LOQ-15IRX9:~$ sudo tcpdump -i any -n port 8080
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
22:57:48.613628 lo    In  IP 127.0.0.1.54662 > 127.0.0.1.8080: Flags [S], seq 1122666259, win 65495, options [mss 65495,sackOK,TS val 4167578184 ecr 0,nop,wscale 10], length 0
22:57:49.661238 lo    In  IP 127.0.0.1.54662 > 127.0.0.1.8080: Flags [S], seq 1122666259, win 65495, options [mss 65495,sackOK,TS val 4167579232 ecr 0,nop,wscale 10], length 0
22:57:50.685354 lo    In  IP 127.0.0.1.54662 > 127.0.0.1.8080: Flags [S], seq 1122666259, win 65495, options [mss 65495,sackOK,TS val 4167580256 ecr 0,nop,wscale 10], length 0
```

В выводе tcpdump наблюдались только исходящие SYN-пакеты от клиента.
Ответные пакеты SYN-ACK от сервера отсутствуют. Это подтверждает, что входящие пакеты на порт 8080 отбрасываются правилом iptables ещё до того, как достигают приложения Python.


## Задание 4. Аппаратное ускорение сетевого трафика (offloading)
### Шаг 1: Определение имени сетевого интерфейса

```
denich@denich-LOQ-15IRX9:~$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP> 
enp7s0           UP             40:c2:ba:92:d0:d6 <BROADCAST,MULTICAST,UP,LOWER_UP> 
wlp0s20f3        UP             44:a3:bb:6c:b9:93 <BROADCAST,MULTICAST,UP,LOWER_UP> 
br-2de54b778a04  DOWN           f2:03:7c:c3:c2:35 <NO-CARRIER,BROADCAST,MULTICAST,UP> 
br-4eee9b5d3bd6  DOWN           d6:31:79:a8:72:b1 <NO-CARRIER,BROADCAST,MULTICAST,UP> 
br-52d9cd9e3c1f  DOWN           7e:89:2e:52:37:45 <NO-CARRIER,BROADCAST,MULTICAST,UP> 
docker0          DOWN           92:c3:e2:40:2c:7c <NO-CARRIER,BROADCAST,MULTICAST,UP>
```
Основной проводной интерфейс — enp7s0.

### Шаг 2: Информация о драйвере адаптера
```
denich@denich-LOQ-15IRX9:~$ ethtool -i enp7s0
driver: r8169
version: 7.0.0-38-generic
firmware-version: rtl8168h-2_0.0.2 02/26/15
expansion-rom-version: 
bus-info: 0000:07:00.0
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: yes
supports-priv-flags: no
```
Адаптер работает на драйвере r8169, PCI-адрес 0000:07:00.0

### Шаг 3: Исследование offload-возможностей адаптера

```
denich@denich-LOQ-15IRX9:~$ ethtool -k enp7s0
Features for enp7s0:
rx-checksumming: on
tx-checksumming: on
	tx-checksum-ipv4: on
	tx-checksum-ip-generic: off [fixed]
	tx-checksum-ipv6: on
	tx-checksum-fcoe-crc: off [fixed]
	tx-checksum-sctp: off [fixed]
scatter-gather: on
	tx-scatter-gather: on
	tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: off [fixed]
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: on
	tx-tcp-accecn-segmentation: off [fixed]
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: on
tx-vlan-offload: on
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: on [fixed]
rx-vlan-filter: off [fixed]
vlan-challenged: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
tx-gso-list: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off
rx-all: off
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-hw-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
rx-gro-hw: off [fixed]
tls-hw-record: off [fixed]
rx-gro-list: off
macsec-hw-offload: off [fixed]
rx-udp-gro-forwarding: off
hsr-tag-ins-offload: off [fixed]
hsr-tag-rm-offload: off [fixed]
hsr-fwd-offload: off [fixed]
hsr-dup-offload: off [fixed]
```


Ключевые включённые функции offload:
| Функция | Статус | Назначение |
|---|---|---|
| rx-checksumming | on | Проверка контрольных сумм при приёме на NIC |
| tx-checksumming | on | Вычисление контрольных сумм при отправке на NIC |
| tx-checksum-ipv4 / -ipv6 | on | Checksum offload для IPv4/IPv6 |
| scatter-gather (tx-scatter-gather) | on | Сборка пакета из разрозненных буферов памяти без копирования |
| tcp-segmentation-offload | on | TSO — сегментация TCP на NIC |
| generic-segmentation-offload | on | GSO — программная сегментация в стеке |
| generic-receive-offload | on | GRO — объединение входящих пакетов |
| rx-vlan-offload / tx-vlan-offload | on | Обработка VLAN-тегов на NIC |

### Шаг 4: Проверка статуса TCP Segmentation Offload

```
denich@denich-LOQ-15IRX9:~$ ethtool -k enp7s0 | grep -i tcp-segmentation
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
```

TCP Segmentation Offload включён, в том числе для TCP over IPv4 . Дополнительно включён TSO для IPv6 (tx-tcp6-segmentation: on).

### Шаг 5: Задача, которую решает TCP Segmentation Offload

TSO позволяет ОС передать сетевой карте один большой TCP-сегмент, а карта сама нарезает его на пакеты размером MTU. Это убирает с CPU рутинную работу по сегментации, повышая пропускную способность и снижая загрузку процессора при интенсивной отправке данных.


# Домашнее задание 2. WEB. Балансировка нагрузки
## Задание 1. Создание Server Pool для backend 

### Шаг 1. Создать две папки в home dir.
```
denich@denich-LOQ-15IRX9:/home$ mkdir -p ~/backend1 ~/backend2
```

### Шаг 2. Наполнить папки содержимым.

```
denich@denich-LOQ-15IRX9:~$ echo '<h1>Response from Backend Server 1</h1>' > ~/backend1/index.html
denich@denich-LOQ-15IRX9:~$ echo '<h2>*** Response from Backend Server 2 ***</h2>' > ~/backend2/index.html
```

Проверим:
```
denich@denich-LOQ-15IRX9:~$ cat ~/backend1/index.html
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ cat ~/backend2/index.html
<h2>*** Response from Backend Server 2 ***</h2>
```

### Шаг 3. Запустить серверы.

```
denich@denich-LOQ-15IRX9:~$ cd ~/backend1 && python3 -m http.server 8081 > /tmp/backend1.log 2>&1 &
[1] 20238
denich@denich-LOQ-15IRX9:~$ cd ~/backend2 && python3 -m http.server 8082 > /tmp/backend2.log 2>&1 &
[2] 20240
```

Проверим, что оба слушают порты:

```
denich@denich-LOQ-15IRX9:~$ ss -lntp | grep -E '8081|8082'
LISTEN 0      5                 0.0.0.0:8082       0.0.0.0:*    users:(("python3",pid=20241,fd=3))    
LISTEN 0      5                 0.0.0.0:8081       0.0.0.0:*    users:(("python3",pid=20239,fd=3))
```

### Шаг 4. Проверим, что каждый отдаёт свой index.html:

```
denich@denich-LOQ-15IRX9:~$ curl -s http://127.0.0.1:8081
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://127.0.0.1:8082
<h2>*** Response from Backend Server 2 ***</h2>
```

## Задание 2. DNS Load Balancing с помощью dnsmasq

### Шаг 1. Настроить конфиг dnsmasq.
```
sudo nano /etc/dnsmasq.d/highload.conf
```

Вставим в конфиг
```
listen-address=127.0.0.1
bind-interfaces

host-record=my-awesome-highload-app.local,127.0.0.1
host-record=my-awesome-highload-app.local,127.0.0.2

no-resolv
```

Проверим, что конфиг записался

```
denich@denich-LOQ-15IRX9:~$ cat /etc/dnsmasq.d/highload.conf
listen-address=127.0.0.1
bind-interfaces

host-record=my-awesome-highload-app.local,127.0.0.1
host-record=my-awesome-highload-app.local,127.0.0.2

no-resolv

```

### Шаг 2. Перезапустить dnsmasq

```
denich@denich-LOQ-15IRX9:~$ sudo systemctl restart dnsmasq
```

### Шаг 3. Проверить порт 53 для DNS

```
denich@denich-LOQ-15IRX9:~$ sudo ss -lntup | grep '127.0.0.1:53'
udp   UNCONN 0      0          127.0.0.1:53         0.0.0.0:*    users:(("dnsmasq",pid=9499,fd=4))                                                                                                                                                                
tcp   LISTEN 0      32         127.0.0.1:53         0.0.0.0:*    users:(("dnsmasq",pid=9499,fd=5))     
```
dnsmasq (PID 9499) слушает 127.0.0.1:53 по UDP и TCP

### Шаг 4. Спросить DNS (backend2 пока работает)

```
denich@denich-LOQ-15IRX9:~$ dig @127.0.0.1 my-awesome-highload-app.local A  

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @127.0.0.1 my-awesome-highload-app.local A
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49972
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;my-awesome-highload-app.local.	IN	A

;; ANSWER SECTION:
my-awesome-highload-app.local. 0 IN	A	127.0.0.1
my-awesome-highload-app.local. 0 IN	A	127.0.0.2

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Wed Oct 28 20:07:39 MSK 2026
;; MSG SIZE  rcvd: 90
```

ANSWER: 2 — обе A-записи на месте. SERVER: 127.0.0.1#53 — ответил наш dnsmasq

### Шаг 5. Остановить backend2

```
denich@denich-LOQ-15IRX9:~$ pkill -f "http.server 8082"
```

### Шаг 6. Снова спросить DNS после отключения backend2

```
denich@denich-LOQ-15IRX9:~$ dig @127.0.0.1 my-awesome-highload-app.local A  

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @127.0.0.1 my-awesome-highload-app.local A
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19016
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;my-awesome-highload-app.local.	IN	A

;; ANSWER SECTION:
my-awesome-highload-app.local. 0 IN	A	127.0.0.2
my-awesome-highload-app.local. 0 IN	A	127.0.0.1

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Wed Oct 28 20:11:18 MSK 2026
;; MSG SIZE  rcvd: 90
```

Проверка:

```
denich@denich-LOQ-15IRX9:~$ curl -s --connect-timeout 2 http://127.0.0.2:8082 || echo "connection failed"
connection failed
```

Обе записи по-прежнему в ответе — dnsmasq не проверяет живость сервисов. Клиент, попавший на 127.0.0.2, получит connection refused. Вывод: DNS-балансировка распределяет запросы, но не даёт отказоустойчивости без health-check.


## Задание 3. Балансировка Layer 4 с  с помощью IPVS

### Шаг 1. Создать dummy-интерфейс

```
denich@denich-LOQ-15IRX9:~$ sudo ip link add name dummy1 type dummy
[sudo] password for denich: 
denich@denich-LOQ-15IRX9:~$ sudo ip link set dummy1 up
denich@denich-LOQ-15IRX9:~$ sudo ip addr add 192.168.100.1/32 dev dummy1
```
Проверка:

```
denich@denich-LOQ-15IRX9:~$ ip addr show dummy1
8: dummy1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 2a:d9:e1:d0:66:28 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/32 scope global dummy1
       valid_lft forever preferred_lft forever
    inet6 fe80::28d9:e1ff:fed0:6628/64 scope link 
       valid_lft forever preferred_lft forever
```

VIP 192.168.100.1/32 успешно назначен на dummy1

### Шаг 2. Настроить IPVS

```
denich@denich-LOQ-15IRX9:~$ sudo ipvsadm -A -t 192.168.100.1:80 -s rr
denich@denich-LOQ-15IRX9:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8081 -m
denich@denich-LOQ-15IRX9:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8082 -m
```

Проверка конфигурации:

```
denich@denich-LOQ-15IRX9:~$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          0         
  -> 127.0.0.1:8082               Masq    1      0          0
```

Создан виртуальный сервер 192.168.100.1:80 с планировщиком rr и два реальных сервера. Режим Masq (NAT) выбран, потому что бэкенды на 127.0.0.1 — IPVS переписывает адрес назначения и отправляет пакеты локально. Вес у обоих 1, то есть нагрузка делится поровну.

### Шаш 3. Демонстрация балансировки через curl

```
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
<h1>Response from Backend Server 1</h1>
denich@denich-LOQ-15IRX9:~$ curl -s http://192.168.100.1
```

ответы строго чередуются — Backend 2, Backend 1, Backend 2, Backend 1… Это наглядная демонстрация round-robin

### Шаг 4. Демонстрация счётчиков IPVS

```
denich@denich-LOQ-15IRX9:~$ sudo ipvsadm -L --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  denich-LOQ-15IRX9:http             13       84       78     5460     7141
  -> localhost:tproxy                    6       39       36     2532     3270
  -> localhost:8082                      7       45       42     2928     3871
```
Счётчики подтверждают балансировку: из 13 соединений 6 ушли на 127.0.0.1:8081 и 7 — на 127.0.0.1:8082. Распределение практически равномерное (6/7), что соответствует алгоритму round-robin. Ненулевые InPkts/OutPkts у обоих бэкендов доказывают, что трафик реально прошёл через оба сервера.

## Задание 4. Балансировка L7 с помощью NGINX

### Шаг 1. Создать конфиг nginx

```
denich@denich-LOQ-15IRX9:~$ cat /etc/nginx/sites-available/highload
server {
    listen 127.0.0.1:8888;
    server_name _;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header X-high-load-test 123;
        proxy_set_header Host $host;
        proxy_next_upstream off;
    }
}

upstream backend_pool {
    zone backend_pool 64k;
    server 127.0.0.1:8081 max_fails=7 fail_timeout=30s;
    server 127.0.0.1:8082 backup;
}
```
nginx слушает только 127.0.0.1:8888. zone backend_pool 64k делает счётчик max_fails общим для всех worker-процессов. max_fails=7 fail_timeout=30s — активный сервер исключается после 7 неудач на 30 секунд. backup — резервный сервер. proxy_next_upstream off запрещает переключение на backup в рамках одного запроса — поэтому клиент видит 502, пока не накопятся 7 неудач. proxy_set_header X-high-load-test 123 добавляет заголовок к запросу в апстрим.

### Шаг 2. Активировать sites и проверить синтаксис

```
denich@denich-LOQ-15IRX9:~$ sudo ln -s /etc/nginx/sites-available/highload /etc/nginx/sites-enabled/highload
denich@denich-LOQ-15IRX9:~$ sudo rm /etc/nginx/sites-enabled/default
denich@denich-LOQ-15IRX9:~$ sudo nginx -t && sudo systemctl restart nginx
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Стандартный сайт default удалён, чтобы nginx не слушал порт 80 — по заданию он должен слушать только 127.0.0.1:8888. nginx -t подтверждает синтаксис, restart применяет конфиг.

### Шаг 3. Проверка, что nginx слушает только 127.0.0.1:8888

```
denich@denich-LOQ-15IRX9:~$ sudo ss -lntp 'sport = :80 or sport = :8888'
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
LISTEN   0        511      127.0.0.1:8888       0.0.0.0:*           users:(("nginx",pid=15228,fd=5),...)
```

### Шаг 4. Запуск бэкендов

```
denich@denich-LOQ-15IRX9:~$ cd ~/backend2 && python3 -m http.server 8082 > /tmp/backend2.log 2>&1 &
denich@denich-LOQ-15IRX9:~$ ss -lntp | grep -E "8081|8082"
LISTEN 0      5      0.0.0.0:8082      0.0.0.0:*    users:(("python3",pid=15250,fd=3))
```

### Шаг 5. Демонстрация переключения после 7 ответов 500

```
denich@denich-LOQ-15IRX9:~$ for i in {1..9}; do echo -n "$i: "; curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:8888/?test=$i"; done
1: 502
2: 502
3: 502
4: 502
5: 502
6: 502
7: 502
8: 200
9: 200
```

первые 7 запросов вернули 502 Bad Gateway — nginx пытался подключиться к 8081, получал connection refused и, из-за proxy_next_upstream off, не переключался на backup в рамках запроса. После 7-й неудачи (max_fails=7) сервер 8081 был помечен как недоступный на fail_timeout=30s, и запросы 8–9 ушли на backup 8082, вернув 200

### Шаг 6. Проверка ответа от backup-сервера
```
denich@denich-LOQ-15IRX9:~$ curl http://127.0.0.1:8888/
<h2>*** Response from Backend Server 2 ***</h2>
```

### Шаг 7. Демонстрация заголовка X-high-load-test через tshark

```
denich@denich-LOQ-15IRX9:~$ sudo tshark -i lo -f "tcp port 8082" -Y "http.request" -T fields -e http.request.line
Capturing on 'Loopback: lo'
X-high-load-test: 123\r\n,Host: backend_pool\r\n,Connection: close\r\n,User-Agent: curl/8.5.0\r\n,Accept: */*\r\n
```
tshark перехватил HTTP-запросы, которые nginx отправил на backup-сервер 127.0.0.1:8082. В заголовках присутствует X-high-load-test: 123 — nginx проставляет кастомный заголовок при проксировании в апстрим. Заголовок Connection: close — стандартное поведение nginx при работе с upstream по HTTP/1.0, на балансировку не влияет.






