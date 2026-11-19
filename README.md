# ЛР5. Мультивендорная корпоративная сеть: MikroTik CHR как vCPE

Курс «Виртуализация сетевых функций». Студент: bobrov, N = 82.

Стенд в containerlab: ядро ЦОД на SONiC, два виртуальных пограничных маршрутизатора MikroTik CHR (hq и br), Linux-узлы srv1, pcb и isp. Между dc1 и hq работает eBGP, на hq настроены NAT и фаервол, филиал подключён через WireGuard, hq управляется через REST API, конфигурации хранятся в репозитории.

## Схема

```
srv1 10.82.100.10 -- dc1 (SONiC, AS 65082) -- hq (CHR, AS 65182) -- isp -- br (CHR) -- pcb 10.82.200.10
       eth1/Ethernet0  Ethernet4 10.82.1.1/30   ether2 10.82.1.2/30  ether3 100.64.1.2/30
                                                                       isp: 100.64.1.1, 100.64.2.1, lo 198.51.100.1
hq <== WireGuard wg-br 10.82.254.1/30 <-> wg-hq 10.82.254.2/30, UDP 13082 ==> br
```

## Адресный план

| Сегмент | Адреса |
|---|---|
| ЦОД | 10.82.100.0/24 (dc1 .1, srv1 .10) |
| Филиал | 10.82.200.0/24 (br .1, pcb .10) |
| dc1 - hq | 10.82.1.0/30 (dc1 .1, hq .2) |
| WAN hq | 100.64.1.0/30 (isp .1, hq .2) |
| WAN br | 100.64.2.0/30 (isp .1, br .2) |
| WireGuard | 10.82.254.0/30 (hq .1, br .2), UDP 13082 |
| AS | dc1 65082, hq 65182 |
| Management | 10.105.82.0/24 |

Устройства: `hq-bobrov`, `br-bobrov` (RouterOS 7.23.8), dc1 (SONiC-VS, FRR).

## Правила межсетевого экрана hq

| # | Цепочка | Условие | Действие | Назначение |
|---|---|---|---|---|
| 1 | input | established, related | accept | ответы на соединения самого роутера |
| 2 | input | invalid | drop | мусорные пакеты вне состояний |
| 3 | input | icmp | accept | ping и диагностика |
| 4 | input | in ether1 | accept | management containerlab: SSH, REST |
| 5 | input | in ether2, tcp/179 | accept | BGP-сессия с dc1 |
| 6 | input | udp/13082 | accept | WireGuard от филиала |
| 7 | input | in ether3 | drop | всё остальное из WAN |
| 8 | forward | established, related | accept | ответы на разрешённые соединения |
| 9 | forward | invalid | drop | мусорные пакеты |
| 10 | forward | ether2 -> ether3 | accept | ЦОД -> интернет |
| 11 | forward | wg-br -> ether2 | accept | филиал -> ЦОД |
| 12 | forward | ether2 -> wg-br | accept | ЦОД -> филиал |
| 13 | forward | всё остальное | drop | default deny |
| NAT | srcnat | out ether3 | masquerade | ЦОД выходит в интернет под 100.64.1.2 |

На br: только srcnat out ether2 masquerade.

Порядок важен: правила проверяются сверху вниз. Правила для management (ether1) и WireGuard стоят выше запрещающих.

## Состав репозитория

```
lab5/
  lab5.clab.yml          топология containerlab
  bootstrap.sh           полный запуск: deploy, hq, br, WireGuard, dc1
  apply.sh               настройка dc1 (FRR) после deploy
  configs/
    hq.rsc               экспорт hq (/export terse)
    br.rsc               экспорт br (/export terse)
    dc1-frr.conf         running-config FRR на dc1
    dc1.sh               команды настройки dc1
  scripts/
    rest-demo.sh         GET, PUT, PATCH, DELETE через REST API hq
```

## Требования

- Linux или WSL2 (Ubuntu) с Docker, buildx и containerlab
- образы `sonic-course:202511` (docker-sonic-vs), `lab-tools:1.0` и `vrnetlab/mikrotik_routeros:7.23.8`
- `sshpass`, `jq`

Образ MikroTik собирается из `chr-7.23.8.vmdk` (mikrotik.com) через [vrnetlab](https://github.com/srl-labs/vrnetlab). Без KVM (например, в WSL2 без вложенной виртуализации) CHR запускается в программной эмуляции QEMU: нужно отключить проверку `/dev/kvm` в `common/vrnetlab.py` и использовать CPU `qemu64`.

## Запуск

Два способа: автоматический (`./bootstrap.sh`) и ручной, пошагово, описан ниже. Во всех командах `STUDENT=bobrov`, `N=82`.

### Автоматически

```
export ROS_PASS=<пароль admin>
./bootstrap.sh
```

### Вручную

#### 0. Подготовка окружения (один раз)

```
sudo apt update
sudo apt install -y docker.io docker-buildx jq git make sshpass
sudo usermod -aG docker $USER
bash -c "$(curl -sL https://get.containerlab.dev)"
```

После `usermod` перезайди в терминал.

Образы:

```
docker build -t lab-tools:1.0 images/lab-tools
docker pull <тут вам надо вписать ваш уже публичный docker-sonic-vs, если кто этим будет пользоваться>
docker tag <тут вам надо вписать ваш уже публичный docker-sonic-vs, если кто этим будет пользоваться> sonic-course:202511
```

Образ MikroTik (нужен `chr-7.23.8.vmdk` с mikrotik.com):

```
git clone https://github.com/srl-labs/vrnetlab.git
cp chr-7.23.8.vmdk vrnetlab/mikrotik/routeros/
cd vrnetlab/mikrotik/routeros
sudo make
cd ../../..
docker images | grep mikrotik
```

Если на хосте нет KVM (WSL2 без вложенной виртуализации), перед `make` отключи проверку KVM:

```
sed -i 's|if os.path.exists("/dev/kvm"):|if False:|; s|cpu="host"|cpu="qemu64"|' vrnetlab/common/vrnetlab.py
```

В этом режиме CHR работает в программной эмуляции, загрузка занимает около минуты.

#### 1. Развернуть топологию

```
export STUDENT=bobrov N=82
cd lab5
sudo env STUDENT=$STUDENT N=$N containerlab deploy -t lab5.clab.yml --reconfigure
```

Не используй `sudo -E`: он не передаёт переменные, и лаборатория получит чужое имя и адреса.

Дождись загрузки CHR (до 2 минут), без ключа `-f`:

```
docker logs clab-$STUDENT-l5-hq 2>&1 | grep -i "startup complete"
docker logs clab-$STUDENT-l5-br 2>&1 | grep -i "startup complete"
```

Если строки нет, повтори команду через минуту. Вход на роутеры: `ssh admin@clab-$STUDENT-l5-hq`, пароль `admin`. Сразу после загрузки SSH может отвечать `Permission denied`: подожди и повтори.

#### 2. Имена и адресация

На hq:

```
/system identity set name=hq-bobrov
/ip address add address=10.82.1.2/30 interface=ether2 comment="to DC"
/ip address add address=100.64.1.2/30 interface=ether3 comment=WAN
/ip route add dst-address=0.0.0.0/0 gateway=100.64.1.1 comment=ISP
```

На br:

```
/system identity set name=br-bobrov
/ip address add address=100.64.2.2/30 interface=ether2 comment=WAN
/ip address add address=10.82.200.1/24 interface=ether3 comment=LAN
/ip route add dst-address=0.0.0.0/0 gateway=100.64.2.1 comment=ISP
```

На dc1 (SONiC). В образе нет `sudo`, создай заглушку с хоста:

```
docker exec clab-$STUDENT-l5-dc1 sh -c 'printf "#!/bin/sh\nexec \"\$@\"\n" > /usr/local/bin/sudo; chmod +x /usr/local/bin/sudo'
docker exec -it clab-$STUDENT-l5-dc1 bash
```

Внутри dc1:

```
supervisorctl start zebra mgmtd staticd bgpd >/dev/null 2>&1
sudo config interface startup Ethernet0
sudo config interface startup Ethernet4
sudo config interface ip add Ethernet0 10.82.100.1/24
sudo config interface ip add Ethernet4 10.82.1.1/30
ip route del default dev eth0
ping -c 3 10.82.1.2
```

Проверка связности с провайдером (RouterOS):

```
/ping 100.64.1.1 count=3
/ping 100.64.2.1 count=3
```

#### 3. eBGP

На dc1 (в `vtysh`):

```
configure terminal
router bgp 65082
 bgp router-id 10.82.1.1
 no bgp ebgp-requires-policy
 neighbor 10.82.1.2 remote-as 65182
 address-family ipv4 unicast
  network 10.82.100.0/24
 exit-address-family
end
write memory
```

На hq:

```
/routing/bgp/instance add name=hq as=65182
/routing/bgp/connection add name=to-dc instance=hq remote.address=10.82.1.1 remote.as=65082 local.role=ebgp output.default-originate=if-installed
/routing/bgp/session print
```

Проверка на dc1: `vtysh -c "show bgp summary"` и `vtysh -c "show ip route"`, там должен быть `B>* 0.0.0.0/0 via 10.82.1.2`.

#### 4. NAT и межсетевой экран (hq)

```
/ip firewall nat add chain=srcnat out-interface=ether3 action=masquerade comment="inside -> Internet"
/ip firewall filter
add chain=input action=accept connection-state=established,related comment="input: established"
add chain=input action=drop connection-state=invalid
add chain=input action=accept protocol=icmp
add chain=input action=accept in-interface=ether1 comment="mgmt (containerlab)"
add chain=input action=accept in-interface=ether2 protocol=tcp dst-port=179 comment="BGP from DC"
add chain=input action=drop in-interface=ether3 comment="drop other from WAN"
add chain=forward action=accept connection-state=established,related comment="forward: established"
add chain=forward action=drop connection-state=invalid
add chain=forward action=accept in-interface=ether2 out-interface=ether3 comment="DC -> Internet"
add chain=forward action=drop comment="default deny"
```

Правило для `ether1` (management) должно стоять выше запрещающих, иначе потеряешь SSH.

Проверка NAT:

```
docker exec clab-$STUDENT-l5-srv1 curl -s -o /dev/null -w "%{http_code}\n" http://198.51.100.1/
docker logs --tail 2 clab-$STUDENT-l5-isp
```

Ответ 200, в логе isp адрес источника 100.64.1.2.

Проверка политики:

```
docker exec clab-$STUDENT-l5-isp ip route add 10.82.0.0/16 via 100.64.1.2
docker exec clab-$STUDENT-l5-isp ping -c 2 -W 1 10.82.100.10
```

Потери 100%. На hq: `/ip firewall filter print stats`, счётчик `default deny` вырос.

#### 5. WireGuard и анонс филиала

На hq:

```
/interface wireguard add name=wg-br listen-port=13082
/ip address add address=10.82.254.1/30 interface=wg-br
:put [/interface wireguard get wg-br public-key]
```

На br:

```
/interface wireguard add name=wg-hq listen-port=13082
/ip address add address=10.82.254.2/30 interface=wg-hq
:put [/interface wireguard get wg-hq public-key]
```

Скопируй выведенные публичные ключи. На hq подставь ключ br:

```
/interface wireguard peers add interface=wg-br public-key="<ключ br>" endpoint-address=100.64.2.2 endpoint-port=13082 allowed-address=10.82.254.2/32,10.82.200.0/24 persistent-keepalive=25s
/ip route add dst-address=10.82.200.0/24 gateway=wg-br
/ip firewall filter add chain=input action=accept protocol=udp dst-port=13082 comment=WireGuard place-before=[find comment="drop other from WAN"]
/ip firewall filter add chain=forward action=accept in-interface=wg-br out-interface=ether2 comment="branch -> DC" place-before=[find comment="default deny"]
/ip firewall filter add chain=forward action=accept in-interface=ether2 out-interface=wg-br comment="DC -> branch" place-before=[find comment="default deny"]
/ip firewall address-list add list=to-dc address=10.82.200.0/24
/routing/bgp/connection set [find name~"to-dc"] output.network=to-dc
```

На br подставь ключ hq:

```
/interface wireguard peers add interface=wg-hq public-key="<ключ hq>" endpoint-address=100.64.1.2 endpoint-port=13082 allowed-address=10.82.254.1/32,10.82.100.0/24 persistent-keepalive=25s
/ip route add dst-address=10.82.100.0/24 gateway=wg-hq
/ip firewall nat add chain=srcnat out-interface=ether2 action=masquerade comment="branch -> Internet"
```

Правило WireGuard должно стоять до `drop other from WAN`, а правила филиала до `default deny`.

Проверка и доказательство шифрования (перехват запускай во втором окне одновременно с пингом):

```
docker exec clab-$STUDENT-l5-dc1 vtysh -c "show ip route 10.82.200.0/24"
docker exec clab-$STUDENT-l5-pcb ping -c 3 10.82.100.10
docker exec clab-$STUDENT-l5-isp timeout 10 tcpdump -nni eth2 -c 6 udp
```

На hq: `/interface wireguard peers print detail` (свежий `last-handshake`, ненулевые rx и tx).

#### 6. REST API и конфигурация как код

На hq включи сервис и вернись на хост:

```
/ip service set www disabled=no
```

```
export ROS_AUTH=admin:admin
./scripts/rest-demo.sh
```

Выгрузка конфигурации в репозиторий:

```
mkdir -p configs
ssh admin@clab-$STUDENT-l5-hq "/export terse" > configs/hq.rsc
ssh admin@clab-$STUDENT-l5-br "/export terse" > configs/br.rsc
docker exec clab-$STUDENT-l5-dc1 vtysh -c "show running-config" > configs/dc1-frr.conf
grep -r private-key configs/ || echo "приватных ключей нет"
```

Коммиты: `git add . && git commit -m "lab5: add vcpe firewall"`.

## Проверка

```
ssh admin@clab-bobrov-l5-hq "/routing/bgp/session print"
docker exec clab-bobrov-l5-dc1 vtysh -c "show bgp summary"
docker exec clab-bobrov-l5-srv1 curl -s -o /dev/null -w "%{http_code}\n" http://198.51.100.1/
docker exec clab-bobrov-l5-pcb ping -c 3 10.82.100.10
docker exec clab-bobrov-l5-isp timeout 10 tcpdump -nni eth2 -c 6 udp
```

Ожидаемый результат: сессия BGP в состоянии established, на dc1 есть `B>* 0.0.0.0/0` и `10.82.200.0/24`, запрос srv1 возвращает 200 и в логе isp виден адрес 100.64.1.2, пинг филиала проходит, а перехват на isp показывает только UDP между 100.64.2.2:13082 и 100.64.1.2:13082.

REST API:

```
export ROS_AUTH=<пользователь>:<пароль>
./scripts/rest-demo.sh
```

## Секреты

Пароли и приватные ключи WireGuard в репозиторий не кладутся. Пароль для REST и SSH берётся из переменных окружения `ROS_AUTH` и `ROS_PASS`. Экспорты в `configs/` сделаны без `show-sensitive`, приватных ключей в них нет. Файлы `configs/*.secret.rsc` исключены через `.gitignore`.

## Особенности

- В образе docker-sonic-vs нет `sudo`, поэтому на dc1 перед настройкой создаётся обёртка `/usr/local/bin/sudo`.
- RouterOS при повторном создании BGP-соединения добавляет суффикс к имени, поэтому сессия может называться `to-dc-1`.
- Free-лицензия CHR ограничивает исходящую скорость до 1 Мбит/с на интерфейс.
