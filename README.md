# nfv-labs-bobrov

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


```

## Требования

- Linux или WSL2 (Ubuntu) с Docker, buildx и containerlab
- образы `sonic-course:202511` (docker-sonic-vs), `lab-tools:1.0` и `vrnetlab/mikrotik_routeros:7.23.8`
- `sshpass`, `jq`

Образ MikroTik собирается из `chr-7.23.8.vmdk` (mikrotik.com) через [vrnetlab](https://github.com/srl-labs/vrnetlab). Без KVM (например, в WSL2 без вложенной виртуализации) CHR запускается в программной эмуляции QEMU: нужно отключить проверку `/dev/kvm` в `common/vrnetlab.py` и использовать CPU `qemu64`.

## Запуск

```
export ROS_PASS=<пароль admin>
./bootstrap.sh
```

Скрипт разворачивает стенд, ждёт загрузки CHR, настраивает hq и br, генерирует ключи WireGuard и прописывает их пирам, затем настраивает dc1. В конце выводит BGP-сессию, пира WireGuard и пинг `pcb -> srv1`.

Вручную:

```
export STUDENT=bobrov N=82
sudo env STUDENT=$STUDENT N=$N containerlab deploy -t lab5.clab.yml
./apply.sh
```

После ручного `deploy` hq и br загружаются без конфигурации, их нужно настроить отдельно.

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
