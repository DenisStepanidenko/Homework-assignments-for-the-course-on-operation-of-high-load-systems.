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


# Домашнее задание 3. Linux. Архитектура и файловые системы

## Задание 1. Kernel and Module Inspection.

### Шаг 1. Версия ядра

```
denich@denich-LOQ-15IRX9:~$ uname -a
Linux denich-LOQ-15IRX9 7.0.0-42-generic #42~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Mon Oct 19 14:43:01 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```

Версия ядра: 7.0.0-42-generic

### Шаг 2. Все загруженные модули ядра

```
denich@denich-LOQ-15IRX9:~$ lsmod | head -15
Module                  Size  Used by
hid_sensor_custom      28672  0
hid_sensor_hub         28672  1 hid_sensor_custom
hidp                   40960  0
nft_reject_ipv4        12288  2
nf_reject_ipv4         12288  1 nft_reject_ipv4
nft_reject             12288  1 nft_reject_ipv4
nft_ct                 28672  1
nft_masq               12288  1
nf_nat_h323            24576  0
nf_conntrack_h323      90112  1 nf_nat_h323
nf_nat_pptp            16384  0
nf_conntrack_pptp      24576  1 nf_nat_pptp
nf_nat_tftp            12288  0
nf_conntrack_tftp      20480  1 nf_nat_tftp
```

Полный список большой, поэтому приведены первые 15 строк

### Шаг 3. Отключение автозагрузки модуля cdrom.

```
denich@denich-LOQ-15IRX9:~$ lsmod | grep cdrom
denich@denich-LOQ-15IRX9:~$ modinfo cdrom | head -5
name:           cdrom
filename:       (builtin)
license:        GPL
file:           drivers/cdrom/cdrom
description:    Uniform CD-ROM driver
```

Модуль cdrom встроен в ядро (filename: (builtin)) и не отображается в lsmod как загружаемый модуль. Это означает, что он присутствует в ядре постоянно и не может быть выгружен или отключён через modprobe.d — механизм blacklist к нему неприменим

### Шаг 4. Конфигурация ядра и параметр CONFIG_XFS_FS

```
denich@denich-LOQ-15IRX9:~$ grep CONFIG_XFS_FS /boot/config-$(uname -r)
CONFIG_XFS_FS=m
denich@denich-LOQ-15IRX9:~$ zcat /proc/config.gz | grep CONFIG_XFS_FS
gzip: /proc/config.gz: No such file or directory
```

Файл конфигурации ядра расположен по пути /boot/config-7.0.0-42-generic. Параметр CONFIG_XFS_FS=m означает, что поддержка файловой системы XFS собрана как загружаемый модуль, а не встроена в ядро. Модуль загружается автоматически при первом монтировании XFS-раздела или вручную через modprobe xfs. Проверка через /proc/config.gz невозможна, так как в ядрах Ubuntu опция CONFIG_IKCONFIG_PROC по умолчанию не включена. Конфигурация ядра успешно получена из файла /boot/config-7.0.0-42-generic, что является штатным способом для данного дистрибутива

## Задание 2. Наблюдение за VFS.

### Шаг 1. Запуск strace

```
denich@denich-LOQ-15IRX9:~$ strace -e trace=openat,read,write,close cat /etc/os-release > /dev/null
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
close(3)                                = 0
openat(AT_FDCWD, "/etc/os-release", O_RDONLY) = 3
read(3, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 131072) = 400
write(1, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 400) = 400
read(3, "", 131072)                     = 0
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
+++ exited with 0 +++
```

strace с фильтром -e trace=openat,read,write,close показывает только системные вызовы работы с файлами: открытие, чтение, запись и закрытие. Перенаправление > /dev/null отбрасывает содержимое файла, но strace пишет в stderr, поэтому его вывод виден.

### Шаг 2. Анализ открываемого и читаемого файла

Открытие /etc/os-release:

```
openat(AT_FDCWD, "/etc/os-release", O_RDONLY) = 3
```

openat открывает файл /etc/os-release в режиме только чтение (O_RDONLY) и возвращает файловый дескриптор 3. Флаг O_RDONLY означает, что файл открывается для чтения, без права записи, cat не собирается ничего записывать в файл.

Чтение файла:

```
read(3, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 131072) = 400
read(3, "", 131072)                     = 0
```

read читает из дескриптора 3 до 131072 байт. Первый вызов вернул 400 байт — это содержимое файла (PRETTY_NAME="Ubuntu 24.04.3 LTS" и остальные строки). Второй вызов вернул 0 — это признак конца файла. cat читает файл до конца, после чего закрывает его.

Запись:

```
write(1, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 400) = 400
```

write пишет 400 байт в дескриптор 1 — это stdout. Поскольку команда запущена с > /dev/null, stdout перенаправлен в /dev/null, и данные фактически отбрасываются. Это не запись в файл на диске — дескриптор 1 указывает на /dev/null, а не на /etc/os-release или другой файл

Закрытие:

```
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
```

cat закрывает дескриптор 3 (/etc/os-release), затем 1 (stdout) и 2 (stderr). Это штатное завершение работы программы.


### Шаг 3. Объяснение отсутствия записывающих вызовов в файл

В выводе нет вызовов write с дескриптором, указывающим на /etc/os-release или другой файл на диске. Единственный write идёт в дескриптор 1 (stdout), который перенаправлен в /dev/null.

Одна причина: cat только читает входные файлы и выводит их в stdout. Он не предназначен для записи в файлы, поэтому и не делает этого.

Остальное — следствия этой причины, а не отдельные причины:   
1) Файл /etc/os-release открыт с флагом O_RDONLY — ядро физически не позволит записать в него данные через этот дескриптор.   
2) write(1, ...) — это запись в stdout. Поскольку stdout перенаправлен в /dev/null, данные никуда не сохраняются. Это не запись в файл в смысле изменения данных на диске.


## Задание 3. LVM Management.

Задание выполнялось на физической машине, а не в виртуальной среде, поэтому добавление второго физического диска /dev/sdb невозможно. Для имитации нового диска создан файл-образ размером 2 ГБ и подключён как loop-устройство.

### Шаг 1. Создание "диска" и раздела

```
denich@denich-LOQ-15IRX9:~$ dd if=/dev/zero of=~/fake_disk.img bs=1M count=2048
2048+0 records in
2048+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 0.432778 s, 5.0 GB/s
denich@denich-LOQ-15IRX9:~$ sudo losetup -fP ~/fake_disk.img
denich@denich-LOQ-15IRX9:~$ lsblk | grep loop35
loop35        7:35   0     2G  0 loop
```

dd создаёт файл fake_disk.img размером 2 ГБ. losetup -fP подключает его как loop-устройство и просит ядро просканировать таблицу разделов. Устройство получило имя /dev/loop35, размер 2G — аналог /dev/sdb из задания.

Создан раздел через fdisk:

```
denich@denich-LOQ-15IRX9:~$ sudo fdisk /dev/loop35
...
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-4194303, default 2048): 
Last sector ... (default 4194303): 
Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
denich@denich-LOQ-15IRX9:~$ sudo partprobe /dev/loop35
denich@denich-LOQ-15IRX9:~$ lsblk /dev/loop35
NAME       MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
loop35       7:35   0   2G  0 loop 
└─loop35p1 259:7    0   2G  0 part
```

Cоздан первичный раздел /dev/loop35p1 на весь диск, тип изменён на 8e (Linux LVM). partprobe обновляет таблицу разделов в ядре без перезагрузки.

### Шаг 2. Создание Physical Volume.

```
denich@denich-LOQ-15IRX9:~$ sudo pvcreate /dev/loop35p1
  Physical volume "/dev/loop35p1" successfully created.
denich@denich-LOQ-15IRX9:~$ sudo pvs
  PV            VG Fmt  Attr PSize  PFree 
  /dev/loop35p1    lvm2 ---  <2.00g <2.00g
```

pvcreate помечает раздел как физический том LVM. pvs подтверждает создание: размер <2.00g, свободно <2.00g


### Шаг 3. Создание Volume Group

```
denich@denich-LOQ-15IRX9:~$ sudo vgcreate vg_highload /dev/loop35p1
  Volume group "vg_highload" successfully created.
denich@denich-LOQ-15IRX9:~$ sudo vgs
  VG          #PV #LV #SN Attr   VSize  VFree 
  vg_highload   1   0   0 wz--n- <2.00g <2.00g
```

vgcreate объединяет PV в пул хранения vg_highload. vgs показывает: 1 PV, 0 LV, размер <2.00g, свободно <2.00g


### Шаг 4. Создание Logical Volume

```
denich@denich-LOQ-15IRX9:~$ sudo lvcreate -L 1200M -n data_lv vg_highload
  Logical volume "data_lv" created.
denich@denich-LOQ-15IRX9:~$ sudo lvcreate -l 100%FREE -n logs_lv vg_highload
  Logical volume "logs_lv" created.
denich@denich-LOQ-15IRX9:~$ sudo lvs
  LV      VG          Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data_lv vg_highload -wi-a-----   1.17g                                                    
  logs_lv vg_highload -wi-a----- 844.00m
```

data_lv создан размером 1200 MiB (LVM отображает как 1.17g). logs_lv создан с -l 100%FREE — забрал всё оставшееся место (844.00m). Сумма 1.17g + 844m ≈ 2G — весь объём VG распределён

### Шаг 5. Форматирование и монтирование data_lv

```
denich@denich-LOQ-15IRX9:~$ sudo mkfs.ext4 /dev/vg_highload/data_lv
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 307200 4k blocks and 76800 inodes
Filesystem UUID: c9c9a7ab-99e6-41a5-a044-35125eb068a8
...
denich@denich-LOQ-15IRX9:~$ sudo mkdir -p /mnt/app_data
denich@denich-LOQ-15IRX9:~$ sudo mount /dev/vg_highload/data_lv /mnt/app_data
denich@denich-LOQ-15IRX9:~$ df -h /mnt/app_data
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/vg_highload-data_lv  1.2G   24K  1.1G   1% /mnt/app_data
```

mkfs.ext4 создаёт файловую систему ext4 на data_lv. Том примонтирован в /mnt/app_data, доступно 1.1G из 1.2G

### Шаг 6. Форматирование и монтирование logs_lv

```
denich@denich-LOQ-15IRX9:~$ sudo apt install -y xfsprogs
...
denich@denich-LOQ-15IRX9:~$ sudo mkfs.xfs /dev/vg_highload/logs_lv
meta-data=/dev/vg_highload/logs_lv isize=512    agcount=4, agsize=54016 blks
...
data     =                       bsize=4096   blocks=216064, imaxpct=25
...
denich@denich-LOQ-15IRX9:~$ sudo mkdir -p /mnt/app_logs
denich@denich-LOQ-15IRX9:~$ sudo mount /dev/vg_highload/logs_lv /mnt/app_logs
denich@denich-LOQ-15IRX9:~$ df -h /mnt/app_logs
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/vg_highload-logs_lv  780M   48M  733M   7% /mnt/app_logs
```


установлен пакет xfsprogs (утилиты XFS). mkfs.xfs создаёт XFS на logs_lv. Том примонтирован в /mnt/app_logs, доступно 733M из 780M

### Шаг 7. Финальная проверка

```
denich@denich-LOQ-15IRX9:~$ df -hT /mnt/app_data /mnt/app_logs
Filesystem                      Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg_highload-data_lv ext4  1.2G   24K  1.1G   1% /mnt/app_data
/dev/mapper/vg_highload-logs_lv xfs   780M   48M  733M   7% /mnt/app_logs
denich@denich-LOQ-15IRX9:~$ lsblk /dev/loop35
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop35                    7:35   0    2G  0 loop 
└─loop35p1              259:7    0    2G  0 part 
  ├─vg_highload-data_lv 252:0    0  1.2G  0 lvm  /mnt/app_data
  └─vg_highload-logs_lv 252:1    0  844M  0 lvm  /mnt/app_logs
```

df -hT подтверждает типы файловых систем: ext4 для /mnt/app_data и xfs для /mnt/app_logs. lsblk показывает полную цепочку: файл-образ loop35 → раздел loop35p1 → LVM-тома data_lv и logs_lv, оба примонтированы в нужные точки


## Задание 4. Использование pseudo filesystem

### Шаг 1. Модель CPU из /proc

```
denich@denich-LOQ-15IRX9:~$ grep "model name" /proc/cpuinfo | head -1
model name	: 13th Gen Intel(R) Core(TM) i7-13650HX
```

/proc/cpuinfo — виртуальный файл, который ядро генерирует при чтении. Поле model name содержит модель процессора: 13th Gen Intel(R) Core(TM) i7-13650HX. head -1 оставлен, потому что cpuinfo содержит блок информации для каждого логического ядра — первая строка соответствует первому ядру

### Шаг 2. Объём памяти (KiB) из /proc

```
denich@denich-LOQ-15IRX9:~$ grep MemTotal /proc/meminfo
MemTotal:       24302132 kB
```

/proc/meminfo — виртуальный файл с информацией о памяти. MemTotal — общий объём доступной RAM: 24 302 132 KiB (≈ 23.2 GiB)


### Шаг 3. PPid текущего shell через /proc/$$/status

```
denich@denich-LOQ-15IRX9:~$ grep PPid /proc/$$/status
PPid:	17324
```

$$ — специальная переменная bash, содержащая PID текущего shell (процесса, в котором выполняется команда). /proc/$$/status — виртуальный файл с информацией об этом процессе. Поле PPid — Parent Process ID, то есть PID родительского процесса (терминала или SSH-сессии), который запустил shell: 17324
$$: это PID текущего экземпляра shell. При каждом новом запуске bash (новый терминал, новая сессия) $$ меняется. В отличие от $! или $?, $$ всегда указывает на сам shell

### Шаг 4. I/O scheduler для основного диска

```
denich@denich-LOQ-15IRX9:~$ cat /sys/block/sda/queue/scheduler
cat: /sys/block/sda/queue/scheduler: No such file or directory
denich@denich-LOQ-15IRX9:~$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline
```

в системе нет диска /dev/sda — основной накопитель это NVMe (nvme0n1). Для NVMe файл планировщика находится по пути /sys/block/nvme0n1/queue/scheduler. Активный планировщик заключён в квадратные скобки: [none], доступен также mq-deadline

### Шаг 5. MTU основного сетевого интерфейса

```
denich@denich-LOQ-15IRX9:~$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp7s0           UP             40:c2:ba:92:d0:d6 <BROADCAST,MULTICAST,UP,LOWER_UP>
wlp0s20f3        UP             44:a3:bb:6c:b9:93 <BROADCAST,MULTICAST,UP,LOWER_UP>
docker0          DOWN           2e:d4:2b:2c:35:8a <NO-CARRIER,BROADCAST,MULTICAST,UP>
br-2de54b778a04  DOWN           32:7f:30:8b:78:d5 <NO-CARRIER,BROADCAST,MULTICAST,UP>
br-4eee9b5d3bd6  DOWN           0e:9d:38:69:26:ed <NO-CARRIER,BROADCAST,MULTICAST,UP>
br-52d9cd9e3c1f  DOWN           36:bf:54:26:fd:dd <NO-CARRIER,BROADCAST,MULTICAST,UP>
denich@denich-LOQ-15IRX9:~$ ip link show enp7s0 | grep mtu
2: enp7s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
denich@denich-LOQ-15IRX9:~$ cat /sys/class/net/enp7s0/mtu
1500
```

основной проводной интерфейс — enp7s0 (состояние UP, есть MAC-адрес). MTU для него — 1500 байт, стандартное значение для Ethernet. Это подтверждается двумя способами: через ip link show (поле mtu 1500) и через sysfs (/sys/class/net/enp7s0/mtu)


# Домашнее задание 4. Linux работа с памятью и процессами

## Задание 1. Systemd

### Шаг 1. Создание bash-скрипта /usr/local/bin/homework_service.sh

```
denich@denich-LOQ-15IRX9:~$ sudo nano /usr/local/bin/homework_service.sh
denich@denich-LOQ-15IRX9:~$ sudo chmod +x /usr/local/bin/homework_service.sh
denich@denich-LOQ-15IRX9:~$ cat /usr/local/bin/homework_service.sh
#!/bin/bash
echo "My custom service has started."
while true; do
  echo "Service heartbeat: $(date)" >> /tmp/homework_service.log
  sleep 15
done
denich@denich-LOQ-15IRX9:~$ ls -l /usr/local/bin/homework_service.sh
-rwxr-xr-x 1 root root 146 Nov 11 20:37 /usr/local/bin/homework_service.sh
```
Скрипт создан, сделан исполняемым, права -rwxr-xr-x подтверждают это. Скрипт выводит приветствие и в бесконечном цикле пишет heartbeat в /tmp/homework_service.log каждые 15 секунд.

### Шаг 2. Создание systemd unit-файла

```
denich@denich-LOQ-15IRX9:~$ sudo nano /etc/systemd/system/homework_service.service
denich@denich-LOQ-15IRX9:~$ cat /etc/systemd/system/homework_service.service
[Unit]
Description=Homework custom service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/homework_service.sh
Restart=on-failure
RestartSec=15

[Install]
WantedBy=multi-user.target
```
unit-файл размещён в /etc/systemd/system/ — это стандартное место для администраторских unit'ов, которое не затирается обновлениями пакетов. Restart=on-failure указывает systemd перезапускать сервис при падении, RestartSec=15 — через 15 секунд. WantedBy=multi-user.target означает, что сервис будет автоматически запускаться при загрузке системы

### Шаг 3. Запуск сервиса и проверка работы

```
denich@denich-LOQ-15IRX9:~$ sudo systemctl daemon-reload
denich@denich-LOQ-15IRX9:~$ sudo systemctl enable homework_service
Created symlink /etc/systemd/system/multi-user.target.wants/homework_service.service → /etc/systemd/system/homework_service.service.
denich@denich-LOQ-15IRX9:~$ sudo systemctl start homework_service
denich@denich-LOQ-15IRX9:~$ sudo systemctl status homework_service --no-pager
● homework_service.service - Homework custom service
     Loaded: loaded (/etc/systemd/system/homework_service.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-11-11 20:38:47 MSK; 3s ago
   Main PID: 13327 (homework_servic)
      Tasks: 2 (limit: 25041)
     Memory: 664.0K (peak: 1.2M)
        CPU: 7ms
     CGroup: /system.slice/homework_service.service
             ├─13327 /bin/bash /usr/local/bin/homework_service.sh
             └─13329 sleep 15

Nov 11 20:38:47 denich-LOQ-15IRX9 systemd[1]: Started homework_service.servi…ce.
Nov 11 20:38:47 denich-LOQ-15IRX9 homework_service.sh[13327]: My custom servi…d.
```

daemon-reload перечитывает unit-файлы. enable создаёт симлинк для автозапуска при загрузке. status показывает: сервис active, Main PID: 13327, процесс /bin/bash /usr/local/bin/homework_service.sh и его дочерний sleep 15. Вывод «My custom service has started.» подтверждает, что скрипт стартовал

### Шаг 4. Проверка записи в лог

```
denich@denich-LOQ-15IRX9:~$ sleep 5
denich@denich-LOQ-15IRX9:~$ cat /tmp/homework_service.log
Service heartbeat: Wed Nov 11 08:38:47 PM MSK 2026
Service heartbeat: Wed Nov 11 08:39:02 PM MSK 2026
```

в логе две записи с интервалом 15 секунд (20:38:47 и 20:39:02) — скрипт работает в цикле и пишет heartbeat каждые 15 секунд, как задано

### Шаг 5. Проверка автоперезапуска при падении

```
denich@denich-LOQ-15IRX9:~$ sudo systemctl show homework_service -p MainPID
MainPID=13327
denich@denich-LOQ-15IRX9:~$ sudo kill -9 13327
denich@denich-LOQ-15IRX9:~$ sleep 20
denich@denich-LOQ-15IRX9:~$ sudo systemctl status homework_service --no-pager
● homework_service.service - Homework custom service
     Loaded: loaded (/etc/systemd/system/homework_service.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-11-11 20:39:44 MSK; 11s ago
   Main PID: 13501 (homework_servic)
      Tasks: 2 (limit: 25041)
     Memory: 584.0K (peak: 1.5M)
        CPU: 6ms
     CGroup: /system.slice/homework_service.service
             ├─13501 /bin/bash /usr/local/bin/homework_service.sh
             └─13503 sleep 15

Nov 11 20:39:44 denich-LOQ-15IRX9 systemd[1]: homework_service.service: Sche… 1.
Nov 11 20:39:44 denich-LOQ-15IRX9 systemd[1]: Started homework_service.servi…ce.
Nov 11 20:39:44 denich-LOQ-15IRX9 homework_service.sh[13501]: My custom servi…d.
```

После kill -9 13327 сервис упал. Systemd, согласно Restart=on-failure и RestartSec=15, перезапустил его через 15 секунд: Main PID сменился с 13327 на 13501, статус снова active, в логе systemd появилась запись Scheduled restart job (обрезано как Sche… 1). Это доказывает, что автоперезапуск работает

### Шаг 6. Топ-5 unit'ов по времени старта

```
denich@denich-LOQ-15IRX9:~$ systemd-analyze blame | head -5
6min 32.985s apt-daily-upgrade.service
     15.219s homework_service.service
     11.385s plymouth-quit-wait.service
      5.873s NetworkManager-wait-online.service
      4.751s docker.service
```

systemd-analyze blame показывает время, которое каждый unit потратил на инициализацию при загрузке. Топ-5:

1) apt-daily-upgrade.service — 6 мин 33 с (обновления пакетов, самый долгий);
2) homework_service.service — 15.2 с (наш сервис);
3) plymouth-quit-wait.service — 11.4 с (загрузочная заставка);
4) NetworkManager-wait-online.service — 5.9 с (ожидание сети);
5) docker.service — 4.8 с.




