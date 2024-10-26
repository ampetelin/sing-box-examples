<h2 align="center">Ubuntu TProxy Gateway</h2>

### Применяемое оборудование и версии ПО
- Оборудование: Raspberry Pi 4
- Операционная система: Ubuntu Server 24.04 LTS

### Настройка ОС
Включаем переадресацию пакетов по протоколу IPv4 и отключаем IPv6, внося изменения в файл `/etc/sysctl.conf` 
```shell
sudo nano /etc/sysctl.conf
```
Раскомментируем строку `net.ipv4.ip_forward=1` для включения переадресации 
```text
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1
```
Добавляем строки для отключения IPv6:
```shell
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```
Применяем изменения
```shell
sudo sysctl -p
```
Создаем таблицу маршрутизации, добавив `100 tproxy` в файл `/etc/iproute2/rt_tables`
```shell
sudo nano /etc/iproute2/rt_tables
```
```text
100     tproxy
255     local
254     main
253     default
0       unspec
```

### Установка sing-box
Устанавливаем sing-box (см. [package-manager](https://sing-box.sagernet.org/installation/package-manager/))
```shell
sudo curl -fsSL https://sing-box.app/gpg.key -o /etc/apt/keyrings/sagernet.asc
sudo chmod a+r /etc/apt/keyrings/sagernet.asc
echo "deb [arch=`dpkg --print-architecture` signed-by=/etc/apt/keyrings/sagernet.asc] https://deb.sagernet.org/ * *" | \
  sudo tee /etc/apt/sources.list.d/sagernet.list > /dev/null
sudo apt-get update
sudo apt-get install sing-box # or sing-box-beta
```

### Настройка sing-box
Редактируем файл конфигурации `sing-box` 
```shell
sudo nano /etc/sing-box/config.json
```
Полностью готовый файл конфигурации с подключением к VLESS-Reality можно позаимствовать из [ubuntu-tproxy-gateway.json](configs/ubuntu-tproxy-gateway.json), который проксирует домен `ifconfig.me` для проверки работоспособности маршрутизации
> [!IMPORTANT]
> Не забудьте изменить следующие конфиденциальные данные подключения к вашему серверу VLESS-Reality:
> - `%server_address%`
> - `%vless_uuid%`
> - `%SNI%`
> - `%uTLS%`
> - `%public_key%`
> - `%short_id%`

> [!WARNING]
> Приведенный в качестве примера конфигурационный файл проксирует только домен ifconfig.me!
> 
> Если вам необходимо добавить свои домены для проксирования, следует ознакомиться с [документацией конфигурирования правил маршрутизации](https://sing-box.sagernet.org/configuration/route/rule/)

Проверяем валидность конфигурационного файла
```shell
sing-box -c /etc/sing-box/config.json check
```
Если ошибок нет, запускаем службу sing-box
```shell
sudo systemctl start sing-box
```
Включаем автозапуск службы sing-box
```shell
sudo systemctl enable sing-box
```

### Настройка Firewall & Routing
Маркируем пакеты, которые будут приходить от устройств, использующих Raspberry Pi в качестве шлюза
```shell
sudo iptables -t mangle -A SINGBOX_TPROXY -d 224.0.0.0/4 -j RETURN 
sudo iptables -t mangle -A SINGBOX_TPROXY -d 255.255.255.255/32 -j RETURN 
sudo iptables -t mangle -A SINGBOX_TPROXY -d 192.168.0.0/16 -p tcp -j RETURN
sudo iptables -t mangle -A SINGBOX_TPROXY -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN
sudo iptables -t mangle -A SINGBOX_TPROXY -j RETURN -m mark --mark 0xff
sudo iptables -t mangle -A SINGBOX_TPROXY -p tcp -j LOG --log-prefix "SINGBOX_TPROXY_TCP: "
sudo iptables -t mangle -A SINGBOX_TPROXY -p tcp -j TPROXY --on-ip 127.0.0.1 --on-port 6969 --tproxy-mark 1
sudo iptables -t mangle -A SINGBOX_TPROXY -p udp -j LOG --log-prefix "SINGBOX_TPROXY_UDP: "
sudo iptables -t mangle -A SINGBOX_TPROXY -p udp -j TPROXY --on-ip 127.0.0.1 --on-port 6969 --tproxy-mark 1
sudo iptables -t mangle -A PREROUTING -i eth0 -j SINGBOX_TPROXY

sudo iptables -t mangle -N DIVERT
sudo iptables -t mangle -A DIVERT -j MARK --set-mark 1
sudo iptables -t mangle -A DIVERT -j ACCEPT
sudo iptables -t mangle -I PREROUTING -p tcp -m socket -j DIVERT
```
> [!WARNING]
> В параметре -d правил:
> 
> iptables -t mangle -A SINGBOX_TPROXY -d 192.168.0.0/16 -p tcp -j RETURN
> 
> iptables -t mangle -A SINGBOX_TPROXY -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN
> 
> используется адрес локальной сети. Если у вас используется отличная от 192.168.0.0/16, необходимо изменить значение
> 
> В параметре -i правила:
> 
> sudo iptables -t mangle -A PREROUTING -i eth0 -j SINGBOX_TPROXY
> 
> используется название сетевого интерфейса, выступающего в качестве шлюза для других устройств. Если у вас используется отличное от eth0, необходимо изменить значение

Сохраняем правила
```shell
sudo netfilter-persistent save
```
> [!TIP]
> Если на устройстве еще нет `netfilter-persistent`, необходимо установить его:
>
> sudo apt install iptables-persistent

Промаркированные пакеты требуется отправить в `TProxy`, для чего редактируем юнит `sing-box.service`
```shell
sudo nano /usr/lib/systemd/system/sing-box.service
```
Добавляем в `[Service]` следующий `ExecStart` и `ExecStop`:
```shell
ExecStart=/sbin/ip rule add fwmark 1 table tproxy ; /sbin/ip route add local 0.0.0.0/0 dev lo table tproxy
ExecStop=/sbin/ip rule del fwmark 1 table tproxy ; /sbin/ip route del local 0.0.0.0/0 dev lo table tproxy
```
Применяем изменения
```shell
sudo systemctl daemon-reload
```
Перезапускаем службу `sing-box`
```shell
sudo systemctl restart sing-box
```

### Настройка клиентов
Клиентам в локальной сети необходимо указать IP адрес Raspberry Pi в качестве шлюза (обязательно) и DNS сервера (опционально)

В случае указания DNS сервера, UDP пакеты на 53 порт попадут в `TProxy`, перехватятся правилом маршрутизации `sing-box`
```json
{
    "protocol": [
        "dns"
    ],
    "outbound": "dns-out"
}
```
И отрезолвятся сервером, указанным в конфигурации `sing-box`:
```json
{
    "dns": {
        "strategy": "prefer_ipv4",
        "servers": [
            {
                "address": "https://1.1.1.1/dns-query",
                "tag": "cloudflare-doh"
            }
        ]
    }
}
```