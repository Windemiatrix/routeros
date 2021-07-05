# 1. Настройка Mikrotik RouterOS

- [1. Настройка Mikrotik RouterOS](#1-настройка-mikrotik-routeros)
  - [1.1. Disclaimer](#11-disclaimer)
  - [1.2. Настройка временного IP адреса на временном интерфейсе](#12-настройка-временного-ip-адреса-на-временном-интерфейсе)
  - [1.3. VLAN](#13-vlan)
  - [1.4. Bridge](#14-bridge)
  - [1.5. Interface list](#15-interface-list)
  - [1.6. IP адреса](#16-ip-адреса)
  - [1.7. Статическая маршрутизация](#17-статическая-маршрутизация)
  - [1.8. DNS](#18-dns)
  - [1.9. Учим Mikrotik правильно работать с несколькими операторами связи](#19-учим-mikrotik-правильно-работать-с-несколькими-операторами-связи)
  - [1.10. Скрипт для включения/отключения операторов связи](#110-скрипт-для-включенияотключения-операторов-связи)
  - [1.11. Очереди трафика](#111-очереди-трафика)
  - [1.12. Публикация портов](#112-публикация-портов)
  - [1.13. Capsman](#113-capsman)
  - [1.14. SNMP](#114-snmp)
  - [1.15. Время](#115-время)
  - [1.16. Отключаем NAT для протокола SIP](#116-отключаем-nat-для-протокола-sip)
  - [1.17. Сетевой экран](#117-сетевой-экран)
  - [1.18. GRE tunnel](#118-gre-tunnel)
    - [1.18.1. Без шифрования](#1181-без-шифрования)
    - [1.18.2. С шифрованием](#1182-с-шифрованием)
    - [1.18.3. Динамические туннели](#1183-динамические-туннели)
  - [1.19. OSPF](#119-ospf)

## 1.1. Disclaimer

Автор не несет ответственности за причиненный вред в процессе настройки Вашего оборудования.

Содержимое данного документа предназначено его же автору, поэтому перечисленные в данном документе манипуляции Вы выполняете на свой страх и риск.

## 1.2. Настройка временного IP адреса на временном интерфейсе

Данный пункт необходимо выполнить для смены стандартного IP адреса оборудования на IP адрес сети предприятия. При смене IP адреса можно потерять доступ к оборудованию, что приведет к сбросу конфигурации и повторной настройке.

Настройка будет рассмотрена на примере оборудования Mikrotik CCR1036-8G-2S+.

Конфигурация оборудования после сброса:

``` routeros
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/ip address
add address=192.168.88.1/24 comment=defconf interface=ether1 network=192.168.88.0
```

Соответственно, подключаемся к интерфейсу `ether1`.

``` routeros
/ip address
add address=192.168.88.2/24 comment=defconf interface=ether2 network=192.168.88.0
```

Теперь мы можем подключиться к интерфейсу `ether2`.

``` routeros
/ip address print
Flags: X - disabled, I - invalid, D - dynamic
 #   ADDRESS            NETWORK         INTERFACE                                                                                                                                                           
 0   ;;; defconf
     192.168.88.1/24    192.168.88.0    ether1
 1   ;;; defconf
     192.168.88.2/24    192.168.88.0    ether2
```

``` routeros
/ip address remove 0
```

Теперь можно переходить к настройкам сети предприятия на интерфейсе `ether1` на основе VLAN.

## 1.3. VLAN

Наша сеть разбита на несколько несколько логических сегментов:

- `ISP-VGS` - оператор связи 1.
- `ISP-VladInfo` - оператор связи 2.
- `LAN-global` - локальная сеть по умолчанию.
- `LAN-servers` - локальная сеть для серверов.
- `Management` - сеть для обслуживания оборудования.

``` routeros
/interface vlan
add comment=ISP1 interface=ether1 name=ISP-VGS vlan-id=101
add comment=ISP2 interface=ether1 name=ISP-VladInfo vlan-id=102
add interface=ether1 name=LAN-global vlan-id=301
add interface=ether1 name=LAN-servers vlan-id=302
add interface=ether1 name=Management vlan-id=201
```

## 1.4. Bridge

``` routeros
/interface bridge
add name=LAN-global-bridge
add name=LAN-servers-bridge
```

``` routeros
/interface bridge port
add bridge=LAN-global-bridge interface=LAN-global
add bridge=LAN-servers-bridge interface=LAN-servers
```

## 1.5. Interface list

`WAN-checked` понадобится в будущем, когда будем тыкать операторов связи палочкой и отключать мертвых.

``` routeros
/interface list
add name=LAN
add name=WAN
add name=WAN-checked
```

``` routeros
/interface list member
add interface=ISP-VGS list=WAN
add interface=ISP-VladInfo list=WAN
add interface=LAN-global-bridge list=LAN
add interface=LAN-servers-bridge list=LAN
add interface=Management list=LAN
```

Также добавим временно в список интерфейс `ether2`:

``` routeros
/interface list member
add interface=ether2 list=LAN
```

И воспользуемся встроенной функцией обранужения интернета:

``` routeros
/interface detect-internet
set detect-interface-list=all internet-interface-list=WAN-checked
```

## 1.6. IP адреса

``` routeros
/ip address
add address=LLL.LLL.LLL.LX/24 interface=LAN-global-bridge
add address=AAA.AAA.AAA.AX/24 comment=ISP1 interface=ISP-VGS
add address=BBB.BBB.BBB.BX/28 comment=ISP2 interface=ISP-VladInfo
```

``` routeros
/ip firewall address-list
add address=10.0.0.0/8 list=RFC1918
add address=100.64.0.0/10 list=RFC1918
add address=172.16.0.0/12 list=RFC1918
add address=192.168.0.0/16 list=RFC1918
add address=BBB.BBB.BBB.BX list=ISP-BBB.BBB.BBB.BX
add address=AAA.AAA.AAA.AX list=ISP-AAA.AAA.AAA.AX
```

## 1.7. Статическая маршрутизация

``` routeos
/ip route
add distance=1 gateway=AAA.AAA.AAA.AG routing-mark=ISP1
add distance=1 gateway=BBB.BBB.BBB.BG routing-mark=ISP2
add comment=ISP-all distance=1 gateway=AAA.AAA.AAA.AG,BBB.BBB.BBB.BG
add check-gateway=ping comment=ISP1 distance=100 gateway=AAA.AAA.AAA.AG
add check-gateway=ping comment=ISP2 distance=100 gateway=BBB.BBB.BBB.BG

/ip route rule
add action=lookup-only-in-table src-address=AAA.AAA.AAA.AX/32 table=ISP1
add action=lookup-only-in-table src-address=BBB.BBB.BBB.BX/32 table=ISP2
```

## 1.8. DNS

``` routeros
/ip dns
set allow-remote-requests=yes servers=8.8.8.8,77.88.8.1
```

## 1.9. Учим Mikrotik правильно работать с несколькими операторами связи

Почему так - радостно ищем в интернете.

``` routeros
/ip firewall mangle
add action=mark-connection chain=prerouting connection-mark=no-mark in-interface=ISP-VGS new-connection-mark=con-ISP1 passthrough=yes
add action=mark-connection chain=prerouting connection-mark=no-mark in-interface=ISP-VladInfo new-connection-mark=con-ISP2 passthrough=yes
add action=mark-routing chain=prerouting connection-mark=con-ISP1 in-interface-list=!WAN new-routing-mark=ISP1 passthrough=yes
add action=mark-routing chain=prerouting connection-mark=con-ISP2 in-interface-list=!WAN new-routing-mark=ISP2 passthrough=yes
add action=mark-routing chain=prerouting connection-mark=con-out-ISP1 in-interface-list=!WAN new-routing-mark=ISP1 passthrough=yes
add action=mark-routing chain=prerouting connection-mark=con-out-ISP2 in-interface-list=!WAN new-routing-mark=ISP2 passthrough=yes
add action=mark-connection chain=forward connection-mark=no-mark new-connection-mark=con-out-ISP1 out-interface=ISP-VGS passthrough=yes
add action=mark-connection chain=forward connection-mark=no-mark new-connection-mark=con-out-ISP2 out-interface=ISP-VladInfo passthrough=yes
add action=mark-connection chain=input in-interface=ISP-VGS new-connection-mark=con-ISP1
add action=mark-connection chain=input in-interface=ISP-VladInfo new-connection-mark=con-ISP2
add action=mark-routing chain=output connection-mark=con-ISP1 new-routing-mark=ISP1 passthrough=yes
add action=mark-routing chain=output connection-mark=con-ISP2 new-routing-mark=ISP2 passthrough=yes
```

При необходимости прибить соединение к конкретному оператору связи:

``` routeros
/ip firewall mangle
add action=mark-routing chain=prerouting dst-address-list=SIP-RTK in-interface-list=LAN new-routing-mark=ISP3 passthrough=yes
add action=mark-routing chain=prerouting dst-port=5000-5003,55777 new-routing-mark=ISP2 passthrough=yes protocol=udp src-address=ttt.ttt.ttt.ttt
```

## 1.10. Скрипт для включения/отключения операторов связи

``` routeros
/system script
add dont-require-permissions=no name=ISP owner=rmartsev policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source="{\r\
    \n    :local ISP [ :toarray \"\" ];\r\
    \n    :local Int \"\";\r\
    \n    :local IP \"\";\r\
    \n    :foreach i in=[ /interface list member print as-value where comment=\"INTERNET detected\" ] do={\r\
    \n        :set IP [/ip address get [find interface=\$i->\"interface\" disabled=no] address];\r\
    \n        :for i from=( [:len \$IP] - 1) to=0 do={\r\
    \n            :if ( [:pick \$IP \$i] = \"/\") do={\r\
    \n                :set IP [:pick \$IP 0 \$i];\r\
    \n            }\r\
    \n        }\r\
    \n        :if ([/ping 8.8.8.8 interval=100ms count=10 src-addres=\$IP] > 8 && [/ping 77.88.8.1 interval=100ms count=10 src-addres=\$IP] > 8) do={\r\
    \n            :set ISP (\$ISP, {[/ip route get [find comment=[/interface get [find name=\$i->\"interface\"] comment]] gateway]});\r\
    \n        } else={\r\
    \n            /ip firewall connection remove [find connection-mark ~ (\".*\" . [/interface get [find name=\$i->\"interface\"] comment])];\r\
    \n        }\r\
    \n    }\r\
    \n    /ip route set [find comment=\"ISP-all\"] gateway=\$ISP;\r\
    \n}\r\
    \n"
/system scheduler
add interval=30s name=ISP on-event=ISP policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon start-time=startup
```

## 1.11. Очереди трафика

Конфигурацию можно сократить, но намеренно этого делать не стал. Возможно, придется что-либо более тонко настроить.

В данном примере описаны конфигурации для настройки очередей для следующих типов трафика:

- веб;
- почта;
- торренты;
- все остальное.

``` routeros
/ip firewall mangle
add action=mark-packet chain=postrouting comment="Download - email" connection-mark=con-out-ISP1 new-packet-mark=packet-email-in-isp1 out-interface-list=LAN packet-mark=no-mark passthrough=yes protocol=tcp src-port=25,587,465,110,143,993,995
add action=mark-packet chain=postrouting connection-mark=con-out-ISP2 new-packet-mark=packet-email-in-isp2 out-interface-list=LAN packet-mark=no-mark passthrough=yes protocol=tcp src-port=25,587,465,110,143,993,995
add action=mark-packet chain=postrouting comment="Upload - email" connection-mark=con-out-ISP1 dst-port=25,587,465,110,143,993,995 new-packet-mark=packet-email-out-isp1 out-interface-list=WAN packet-mark=no-mark passthrough=yes protocol=tcp
add action=mark-packet chain=postrouting connection-mark=con-out-ISP2 dst-port=25,587,465,110,143,993,995 new-packet-mark=packet-email-out-isp2 out-interface-list=WAN packet-mark=no-mark passthrough=yes protocol=tcp
add action=mark-packet chain=postrouting comment="Download - web" connection-mark=con-out-ISP1 new-packet-mark=packet-web-in-isp1 out-interface-list=LAN packet-mark=no-mark passthrough=yes protocol=tcp src-port=80,443,8080
add action=mark-packet chain=postrouting connection-mark=con-out-ISP2 new-packet-mark=packet-web-in-isp2 out-interface-list=LAN packet-mark=no-mark passthrough=yes protocol=tcp src-port=80,443,8080
add action=mark-packet chain=postrouting comment="Upload - web" connection-mark=con-out-ISP1 dst-port=80,443,8080 new-packet-mark=packet-web-out-isp1 out-interface-list=WAN packet-mark=no-mark passthrough=yes protocol=tcp
add action=mark-packet chain=postrouting connection-mark=con-out-ISP2 dst-port=80,443,8080 new-packet-mark=packet-web-out-isp2 out-interface-list=WAN packet-mark=no-mark passthrough=yes protocol=tcp
add action=mark-packet chain=postrouting comment="Download - p2p" dst-address-list=p2p-seeds new-packet-mark=packet-p2p-in packet-mark=no-mark passthrough=yes
add action=mark-packet chain=postrouting comment="Upload - p2p" new-packet-mark=packet-p2p-out packet-mark=no-mark passthrough=yes src-address-list=p2p-seeds
add action=mark-packet chain=postrouting comment="Download - all" connection-mark=con-out-ISP1 new-packet-mark=packet-all-in-isp1 out-interface-list=LAN packet-mark=no-mark passthrough=yes
add action=mark-packet chain=postrouting connection-mark=con-out-ISP2 new-packet-mark=packet-all-in-isp2 out-interface-list=LAN packet-mark=no-mark passthrough=yes
add action=mark-packet chain=postrouting comment="Upload - all" connection-mark=con-out-ISP1 new-packet-mark=packet-all-out-isp1 out-interface-list=WAN packet-mark=no-mark passthrough=yes
add action=mark-packet chain=postrouting connection-mark=con-out-ISP2 new-packet-mark=packet-all-out-isp2 out-interface-list=WAN packet-mark=no-mark passthrough=yes
```

Определение торрент трафика выделим в отдельную конфигурацию. Она подлежит дальнейшей доработке, т.к. в address-list добавляются и локальные и глобальные IP адреса. Хорошо бы их разделить. Однако, в текущем виде конфигурация работает.

Данный набор правил основан на спецификации протоколов передачи данных, изменения стоит вносить только в логику добавления в addredd-list.

``` routeros
/ip firewall layer7-protocol
add name=DHT regexp=^d1:.d2:id20:
add name=BitTorrent regexp="\13bittorrent protocol"
add name=ut_pex regexp=.*:md11:.*:ut_pex.*:
add name="\B5TP_FIN" regexp="^\11\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name="\B5TP_STATE" regexp="^!\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name="\B5TP_RESET" regexp="^1\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name="\B5TP_SYN" regexp="^A\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name=BitTorrent_Teredo regexp="^`.*\13bittorrent protocol"
add name=DHT_Teredo regexp="^`.*d1:[a|r]d2:id20:.*y1:[q|r]e"
add name=ut_pex_Teredo regexp="^`.*:md11:.*:ut_pex.*"
add name="\B5TP_SYN_Teredo" regexp="^`.*A\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name="\B5TP_STATE_Teredo" regexp="^`.*!\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name="\B5TP_RESET_Teredo" regexp="^`.*1\00.{2}.{4}.{4}.{4}.{2}.{2}"
add name="\B5TP_FIN_Teredo" regexp="^`.*\11\00.{2}.{4}.{4}.{4}.{2}.{2}"
/ip firewall mangle
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent ut_pex in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol=ut_pex protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol=ut_pex protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol=ut_pex protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol=ut_pex protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent ut_pex out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol=ut_pex out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol=ut_pex out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol=ut_pex out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol=ut_pex out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent ut_pex_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol=ut_pex_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol=ut_pex_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol=ut_pex_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol=ut_pex_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent ut_pex_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol=ut_pex_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol=ut_pex_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol=ut_pex_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol=ut_pex_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent BitTorrent in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol=BitTorrent protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol=BitTorrent protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol=BitTorrent protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol=BitTorrent protocol=tcp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent BitTorrent out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol=BitTorrent out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol=BitTorrent out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol=BitTorrent out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol=BitTorrent out-interface-list=WAN protocol=tcp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent BitTorrent_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol=BitTorrent_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol=BitTorrent_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol=BitTorrent_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol=BitTorrent_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent BitTorrent_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol=BitTorrent_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol=BitTorrent_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol=BitTorrent_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol=BitTorrent_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent DHT in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol=DHT protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol=DHT protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol=DHT protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol=DHT protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent DHT out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol=DHT out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol=DHT out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol=DHT out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol=DHT out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent DHT_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol=DHT_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol=DHT_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol=DHT_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol=DHT_Teredo protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent DHT_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol=DHT_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol=DHT_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol=DHT_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol=DHT_Teredo out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_FIN in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_FIN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_FIN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_FIN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_FIN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_FIN out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_FIN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_FIN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_FIN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_FIN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_FIN_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_FIN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_FIN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_FIN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_FIN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_FIN_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_FIN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_FIN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_FIN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_FIN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_RESET in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_RESET" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_RESET" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_RESET" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_RESET" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_RESET out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_RESET" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_RESET" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_RESET" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_RESET" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_RESET_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_RESET_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_RESET_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_RESET_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_RESET_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_RESET_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_RESET_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_RESET_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_RESET_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_RESET_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_STATE in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_STATE" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_STATE" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_STATE" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_STATE" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_STATE out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_STATE" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_STATE" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_STATE" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_STATE" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_STATE_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_STATE_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_STATE_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_STATE_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_STATE_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_STATE_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_STATE_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_STATE_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_STATE_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_STATE_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_SYN in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_SYN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_SYN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_SYN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_SYN" packet-size=48-54 protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_SYN out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_SYN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_SYN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_SYN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_SYN" out-interface-list=WAN packet-size=48-54 protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_SYN_Teredo in - add peers to address list" connection-mark=con-out-ISP1 in-interface-list=WAN layer7-protocol="\B5TP_SYN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 in-interface-list=WAN layer7-protocol="\B5TP_SYN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 in-interface-list=WAN layer7-protocol="\B5TP_SYN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 in-interface-list=WAN layer7-protocol="\B5TP_SYN_Teredo" protocol=udp src-address-list=!p2p-seeds src-port=1024-65535
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward comment="Torrent \B5TP_SYN_Teredo out - add peers to address list" connection-mark=con-out-ISP1 dst-port=1024-65535 layer7-protocol="\B5TP_SYN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP2 dst-port=1024-65535 layer7-protocol="\B5TP_SYN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP3 dst-port=1024-65535 layer7-protocol="\B5TP_SYN_Teredo" out-interface-list=WAN protocol=udp
add action=add-src-to-address-list address-list=p2p-seeds address-list-timeout=none-dynamic chain=forward connection-mark=con-out-ISP4 dst-port=1024-65535 layer7-protocol="\B5TP_SYN_Teredo" out-interface-list=WAN protocol=udp
add action=mark-connection chain=forward comment="Marking torrent connections and packets" new-connection-mark=con-p2p-out passthrough=yes src-address-list=p2p-seeds
add action=mark-connection chain=forward dst-address-list=p2p-seeds new-connection-mark=con-p2p-in passthrough=yes
```

``` routeros
/queue tree
add max-limit=500M name="Global download" parent=global
add max-limit=500M name="Global upload" parent=global
add limit-at=10M max-limit=100M name=7-in-all packet-mark=packet-all-in-isp1,packet-all-in-isp2,packet-all-in-isp3,packet-all-in-isp4 parent="Global download" priority=7 queue=pcq-download-default
add limit-at=10M max-limit=100M name=7-out-all packet-mark=packet-all-out-isp1,packet-all-out-isp2,packet-all-out-isp3,packet-all-out-isp4 parent="Global upload" priority=7 queue=pcq-upload-default
add limit-at=20M max-limit=60M name=2-in-trueconf packet-mark=packet-trueconf-in-isp1,packet-trueconf-in-isp2,packet-trueconf-in-isp3,packet-trueconf-in-isp4 parent="Global download" priority=2 queue=pcq-download-default
add limit-at=20M max-limit=60M name=2-out-trueconf packet-mark=packet-trueconf-out-isp1,packet-trueconf-out-isp2,packet-trueconf-out-isp3,packet-trueconf-out-isp4 parent="Global upload" priority=2 queue=pcq-upload-default
add limit-at=50M max-limit=300M name=6-in-web packet-mark=packet-web-in-isp1,packet-web-in-isp2,packet-web-in-isp3,packet-web-in-isp4 parent="Global download" priority=6 queue=pcq-download-default
add limit-at=50M max-limit=300M name=6-out-web packet-mark=packet-web-out-isp1,packet-web-out-isp2,packet-web-out-isp3,packet-web-out-isp4 parent="Global upload" priority=6 queue=pcq-upload-default
add limit-at=20M max-limit=50M name=3-in-email packet-mark=packet-email-in-isp1,packet-email-in-isp2,packet-email-in-isp3,packet-email-in-isp4 parent="Global download" priority=3 queue=pcq-download-default
add limit-at=20M max-limit=50M name=3-out-email packet-mark=packet-email-out-isp1,packet-email-out-isp2,packet-email-out-isp3,packet-email-out-isp4 parent="Global upload" priority=3 queue=pcq-upload-default
add limit-at=5M max-limit=30M name=8-in-p2p packet-mark=packet-p2p-in parent="Global download" queue=pcq-download-default
add limit-at=5M max-limit=30M name=8-out-p2p packet-mark=packet-p2p-out parent="Global upload" queue=pcq-download-default
```

## 1.12. Публикация портов

``` routeros
/ip firewall nat
add action=dst-nat chain=dstnat dst-address-list=ISP-213.108.218.16 dst-port=9020 protocol=tcp to-addresses=192.168.48.20 to-ports=9020
```

## 1.13. Capsman

``` routeros
/caps-man channel
add band=2ghz-g/n frequency=2462 name=2.4G-ch11 tx-power=17
add band=5ghz-n/ac frequency=5320 name=5G-ch64 tx-power=17
/caps-man datapath
add bridge=LAN-global-bridge client-to-client-forwarding=yes name=LAN-global
/caps-man security
add authentication-types=wpa2-psk encryption=aes-ccm group-encryption=aes-ccm name=ESSID1-security passphrase=<PASSWORD>
add authentication-types=wpa2-psk encryption=aes-ccm group-encryption=aes-ccm name=ESSID2-security passphrase=<PASSWORD>
/caps-man configuration
add channel=2.4G-ch11 country=russia datapath=LAN-global guard-interval=any hw-protection-mode=rts-cts hw-retries=7 mode=ap name=ESSID1-2.4G-config rx-chains=0,1,2 security=ESSID1-security ssid=ESSID1 tx-chains=0,1,2
add channel=5G-ch64 country=russia datapath=LAN-global guard-interval=any hw-protection-mode=rts-cts hw-retries=7 mode=ap name=ESSID1-5G-config rx-chains=0,1,2 security=ESSID1-security ssid=ESSID1 tx-chains=0,1,2
add channel=2.4G-ch11 country=russia datapath=LAN-global datapath.interface-list=ESSID2 guard-interval=any hw-protection-mode=rts-cts hw-retries=7 mode=ap name=ESSID2-2.4G-config rx-chains=0,1,2 security=ESSID2-security ssid=ESSID2 tx-chains=0,1,2
add channel=5G-ch64 country=russia datapath=LAN-global datapath.interface-list=ESSID2 guard-interval=any hw-protection-mode=rts-cts hw-retries=7 mode=ap name=ESSID2-5G-config rx-chains=0,1,2 security=ESSID2-security ssid=ESSID2 tx-chains=0,1,2
/caps-man manager
set ca-certificate=auto certificate=auto enabled=yes require-peer-certificate=yes upgrade-policy=suggest-same-version
/caps-man provisioning
add action=create-dynamic-enabled hw-supported-modes=g,gn master-configuration=ESSID1-2.4G-config name-format=prefix-identity name-prefix=2.4G slave-configurations=ESSID2-2.4G-config
add action=create-dynamic-enabled hw-supported-modes=an,ac master-configuration=ESSID1-5G-config name-format=prefix-identity name-prefix=5G slave-configurations=ESSID2-5G-config
```

## 1.14. SNMP

``` routeros
/snmp
set enabled=yes trap-generators=temp-exception,interfaces,start-trap trap-version=2
```

## 1.15. Время

``` routeros
/system clock
set time-zone-name=Europe/Moscow
```

## 1.16. Отключаем NAT для протокола SIP

``` routeros
/ip firewall service-port
set sip disabled=yes sip-timeout=1m
```

## 1.17. Сетевой экран

``` routeros
/ip firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="Drop SSH from blacklist" dst-port=22 protocol=tcp src-address-list=ssh_blacklist
add action=add-src-to-address-list address-list=ssh_blacklist address-list-timeout=1h chain=input connection-state=new dst-port=22 protocol=tcp src-address-list=ssh_stage3
add action=add-src-to-address-list address-list=ssh_stage3 address-list-timeout=15m chain=input connection-state=new dst-port=22 protocol=tcp src-address-list=ssh_stage2
add action=add-src-to-address-list address-list=ssh_stage2 address-list-timeout=10m chain=input connection-state=new dst-port=22 protocol=tcp src-address-list=ssh_stage1
add action=add-src-to-address-list address-list=ssh_stage1 address-list-timeout=5m chain=input connection-state=new dst-port=22 protocol=tcp
add action=accept chain=input connection-state=new dst-port=22 protocol=tcp
add action=accept chain=forward comment="Accept DSTNAT" connection-nat-state=dstnat
add action=accept chain=input comment=Everdo dst-port=5555 protocol=tcp
add action=accept chain=input comment=L2TP/IPsec dst-port=1701,500,4500 log=yes protocol=udp
add action=accept chain=input protocol=ipsec-esp
add action=accept chain=input protocol=l2tp
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
add action=accept chain=input comment="defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
add action=drop chain=input comment="defconf: drop all not coming from LAN" in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept in ipsec policy" ipsec-policy=in,ipsec
add action=accept chain=forward comment="defconf: accept out ipsec policy" ipsec-policy=out,ipsec
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" connection-state=established,related disabled=yes
add action=accept chain=forward comment="defconf: accept established,related, untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=WAN
```

## 1.18. GRE tunnel

### 1.18.1. Без шифрования

Mikrotik:

``` routeros
/interface gre
add mtu=1300 \
    name=GRE_Tunnel_0 \
    local-address=83.97.X.X \
    remote-address=91.195.X.X

/interface list member 
add interface=GRE_Tunnel_0 \
    list=LAN
```

Cisco:

``` IOS
interface Tunnel1
 vrf forwarding LAN
 ip address 10.32.X.X 255.255.255.252
 ip mtu 1300
 ip tcp adjust-mss 1360
 tunnel source 192.168.X.X
 tunnel destination 192.168.X.X
 tunnel path-mtu-discovery
 tunnel vrf 301
```

### 1.18.2. С шифрованием

``` routeros
/ip ipsec profile
add dh-group=modp1024 \
    enc-algorithm=aes-256 \
    hash-algorithm=sha256 \
    lifetime=1h \
    name=profile-aes256-sha256 \
    nat-traversal=no

/ip ipsec proposal
add auth-algorithms=sha256 \
    enc-algorithms=aes-256-cbc,aes-256-ctr,aes-256-gcm \
    lifetime=1h \
    name=proposal-aes256-sha256 \
    pfs-group=none

/ip ipsec peer
add address=10.32.X.X/32 \
    local-address=10.32.X.X \
    name=Peer_1 \
    profile=profile-aes256-sha256

/ip ipsec identity
add peer=Peer_1 \
    secret=<<secret>>

/ip ipsec policy
set 0 disabled=yes
add dst-address=10.32.X.X/32 \
    peer=Peer_1 \
    proposal=proposal-aes256-sha256 \
    protocol=gre \
    src-address=10.32.X.X/32 \
    tunnel=yes
```

### 1.18.3. Динамические туннели

``` routeros
/ip ipsec policy group
add name=Dynamic

/ip ipsec peer
add exchange-mode=aggressive \
    local-address=95.66.x.x \
    name=Dynamic-ISP2 \
    passive=yes \
    profile=profile-aes256-sha256
add exchange-mode=aggressive \
    local-address=83.97.x.x \
    name=Dynamic-ISP1 \
    passive=yes \
    profile=profile-aes256-sha256

/ip ipsec identity
add generate-policy=port-override \
    peer=Dynamic-ISP1 \
    policy-template-group=Dynamic \
    secret=<<secret>>
add generate-policy=port-override \
    peer=Dynamic-ISP2 \
    policy-template-group=Dynamic \
    secret=<<secret>>

/ip ipsec policy
add dst-address=0.0.0.0/0 \
    group=Dynamic \
    proposal=proposal-aes256-sha256 \
    src-address=192.168.x.0/24 \
    template=yes
```

## 1.19. OSPF

Mikrotik:

``` routeros
/ip address
add address=10.32.X.X/30 
    interface=GRE_Tunnel_0

/routing ospf instance
set [ find default=yes ] disabled=yes
add name=instance-ORES \
    out-filter=FILTER-OSPF-ORES-OUT \
    redistribute-connected=as-type-2 \
    redistribute-other-ospf=as-type-2 \
    redistribute-static=as-type-2 \
    router-id=0.0.0.1

/routing ospf area
add instance=instance-ORES \
    name=area-ORES

/routing filter
add action=accept \
    chain=FILTER-OSPF-ORES-OUT \
    prefix=192.168.X.X/X \
    protocol=connect
add action=discard \
    chain=FILTER-OSPF-ORES-OUT \
    protocol=connect,static,ospf

/routing ospf interface
add authentication=md5 \
    authentication-key=<<secret>> \
    interface=GRE_Tunnel_0 \
    network-type=point-to-point

/routing ospf network
add area=area-ORES \
    network=10.32.X.X/30
```

Cisco:

``` IOS
interface Tunnel1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 7 <<secret>>
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10

router ospf 1
 router-id 0.0.0.2
 capability vrf-lite
 redistribute connected
 redistribute static
 redistribute ospf 2
 redistribute bgp 65000
 passive-interface default
 distribute-list FILTER-OSPF-1-OUT out 

ip access-list standard FILTER-OSPF-106-OUT
 10 permit X.X.X.X 0.0.0.255
 ...
 900 deny   any
```
