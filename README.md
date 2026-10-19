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





