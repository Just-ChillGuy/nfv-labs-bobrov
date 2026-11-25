# Лабораторная работа №6. Цепочка сервисных функций и жизненный цикл VNF

Репозиторий с материалами для защиты лабораторной работы по дисциплине «Виртуализация сетевых функций».

## Цель работы

Развернуть NFVI на OVS под управлением OpenDaylight, построить сервисную цепочку FW → Shaper, выполнить жизненный цикл VNF (instantiate, configure, scale-out, heal, terminate) и оценить производительность цепочки.

## Персональные параметры

| Параметр | Значение |
| --- | --- |
| `STUDENT` / `N` | `bobrov` / `82` |
| Management / ODL | `10.106.82.0/24` / `10.106.82.100` |
| client1 / client2 / server | `10.82.6.11` / `10.82.6.12` / `10.82.6.100` |
| Запрещённый порт FW | `8082` |
| Скорость шейпера | `12 Мбит/с` |
| Цепочки | `bobrov-c1`, `bobrov-c2` |

## Содержимое репозитория

- `lab6.clab.yml` -- топология containerlab
- `nfvo.py` -- NFVO: читает `vnffg.yaml` и программирует цепочки через RESTCONF OpenDaylight
- `vnfm.sh` -- VNFM: instantiate, configure, terminate и list экземпляров VNF
- `vnffg.yaml` -- описание цепочек (финальная версия: обе через fw2)
- `descriptors/fw-vnfd.yaml`, `descriptors/shaper-vnfd.yaml` -- дескрипторы VNF
- `vnf/fw-rules.nft` -- правила межсетевого экрана
- `scripts/sw-setup.sh` -- настройка OVS и подключение к контроллеру
- `report.tex` -- отчёт по ГОСТ в исходниках LaTeX
- `screenshots/` -- схема и скриншоты ключевых этапов работы

## Требования

- Windows с WSL2 (Ubuntu) или Linux
- Docker и containerlab 0.82
- Образы `lab-tools:1.0`, `ovs-lab:latest`, `odl:chromium` (собираются из `images/` материалов лабораторной)
- `python3-requests`, `python3-yaml`, `gettext-base`

## Схема NFVI

![Схема NFVI](screenshots/00-scheme.png)

| ETSI NFV | В работе |
| --- | --- |
| NFVI | WSL2, Docker, OVS (userspace datapath) |
| VIM | Docker + containerlab |
| SDN-контроллер / WIM | OpenDaylight (OpenFlow Plugin) |
| VNFM | `vnfm.sh` |
| NFVO | `nfvo.py` |
| VNFD / VNFFGD | `descriptors/*.yaml` / `vnffg.yaml` |
| VNF | fw1, fw2 (nftables), shaper1 (tc tbf) |

## Инструкция по воспроизведению

Все команды выполняются в WSL2 (Ubuntu) из каталога `lab6/`. Команды `vnfm.sh` запускаются через `sudo env`, чтобы передать `STUDENT` и `N`.

### 0. Подготовка

```bash
sudo apt-get install -y python3-requests python3-yaml gettext-base git
export STUDENT=bobrov N=82
cd ~/nfv-labs-$STUDENT/lab6
chmod +x vnfm.sh nfvo.py scripts/sw-setup.sh
sed -i 's/\r$//' vnfm.sh nfvo.py scripts/sw-setup.sh vnf/fw-rules.nft vnffg.yaml lab6.clab.yml
```

Сборка образов (если их ещё нет), `IMG` -- каталог `images` из материалов лабораторной:

```bash
docker build -t lab-tools:1.0 $IMG/lab-tools
docker build -t ovs-lab:latest $IMG/ovs-lab
docker build -t odl:chromium $IMG/odl
```

Проверка окружения:

```bash
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}' | grep -E '^(REPOSITORY|lab-tools|ovs-lab|odl)'
docker version --format 'Docker {{.Server.Version}}'
containerlab version | grep -i version
```

### 1. Развёртывание и контроллер

```bash
cat lab6.clab.yml
cat scripts/sw-setup.sh
sudo env STUDENT=$STUDENT N=$N containerlab deploy -t lab6.clab.yml
docker exec -d clab-$STUDENT-l6-server iperf3 -s
docker exec -d clab-$STUDENT-l6-server python3 -m http.server $((8000 + N))
docker exec -t clab-$STUDENT-l6-odl /opt/odl/bin/client "feature:install odl-openflowplugin-flow-services-rest odl-openflowplugin-app-table-miss-enforcer"
docker exec -i -e ODL_IP=10.106.$N.100 clab-$STUDENT-l6-sw sh < scripts/sw-setup.sh
docker exec clab-$STUDENT-l6-sw ovs-vsctl show | grep -A1 Controller
docker exec clab-$STUDENT-l6-client1 ping -c 2 -W 1 10.$N.6.100
```

Если установка features в ODL не проходит, команду нужно повторить через 10-20 секунд: контроллер ещё запускается. До применения цепочки ping не проходит: OVS в режиме `fail-mode=secure` работает только по потокам.

### 2. Настройка VNF

```bash
cat descriptors/fw-vnfd.yaml
cat vnf/fw-rules.nft
cat descriptors/shaper-vnfd.yaml
cat vnfm.sh
sudo env STUDENT=$STUDENT N=$N ./vnfm.sh configure fw1 fw
sudo env STUDENT=$STUDENT N=$N ./vnfm.sh configure shaper1 shaper
./vnfm.sh list
docker exec clab-$STUDENT-l6-fw1 nft list ruleset
docker exec clab-$STUDENT-l6-shaper1 tc qdisc show dev eth2
```

### 3. Сервисная цепочка

```bash
cat vnffg.yaml
python3 nfvo.py apply vnffg.yaml
docker exec clab-$STUDENT-l6-sw ovs-ofctl -O OpenFlow13 dump-flows br0 | grep priority=300 | sort
python3 nfvo.py show vnffg.yaml
```

Потоки цепочки c1 (на каждую строку два потока: IPv4 и ARP):

| Направление | Поток | in_port → out_port | Матч |
| --- | --- | --- | --- |
| прямое | fwd0 | 1 → 3 | src = 10.82.6.11 |
| прямое | fwd1 | 4 → 7 | src = 10.82.6.11 |
| прямое | fwd2 | 8 → 9 | src = 10.82.6.11 |
| обратное | rev0 | 9 → 8 | dst = 10.82.6.11 |
| обратное | rev1 | 7 → 4 | dst = 10.82.6.11 |
| обратное | rev2 | 3 → 1 | dst = 10.82.6.11 |

Проверка прохождения трафика и работы межсетевого экрана:

```bash
docker exec clab-$STUDENT-l6-client1 ping -c 3 10.$N.6.100
docker exec clab-$STUDENT-l6-client1 curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://10.$N.6.100/
docker exec clab-$STUDENT-l6-client1 curl -s -m 3 -o /dev/null -w 'порт '$((8000 + N))': %{http_code}\n' http://10.$N.6.100:$((8000 + N))/
docker exec clab-$STUDENT-l6-fw1 nft list ruleset | grep counter
```

HTTP проходит (200), порт 8082 блокируется (000).

Проверка шейпера:

```bash
docker exec clab-$STUDENT-l6-client1 iperf3 -c 10.$N.6.100 -t 5
docker exec clab-$STUDENT-l6-client1 iperf3 -c 10.$N.6.100 -t 5 -R
```

### 4. Scale-out

```bash
sudo env STUDENT=$STUDENT N=$N ./vnfm.sh instantiate fw2 fw 5 6
./vnfm.sh list
docker exec clab-$STUDENT-l6-sw ovs-ofctl -O OpenFlow13 show br0 | grep -E '^ [56]\('
sed -i '/-c2"/s/\[fw1, shaper1\]/[fw2, shaper1]/' vnffg.yaml
git diff -- vnffg.yaml
python3 nfvo.py apply vnffg.yaml
docker exec clab-$STUDENT-l6-fw1 nft list ruleset | grep 'allow http'
docker exec clab-$STUDENT-l6-fw2 nft list ruleset | grep 'allow http'
docker exec clab-$STUDENT-l6-client2 curl -s -o /dev/null -w 'client2 HTTP %{http_code}\n' http://10.$N.6.100/
docker exec clab-$STUDENT-l6-fw1 nft list ruleset | grep 'allow http'
docker exec clab-$STUDENT-l6-fw2 nft list ruleset | grep 'allow http'
```

Цепочка c2 переведена на `[fw2, shaper1]`, счётчик `allow http` растёт только на fw2.

### 5. Heal и terminate

```bash
date +%T.%N
docker stop clab-$STUDENT-l6-fw1
date +%T.%N
docker exec clab-$STUDENT-l6-client1 ping -c 2 -W 1 10.$N.6.100
docker exec clab-$STUDENT-l6-client2 ping -c 2 10.$N.6.100
sed -i '/-c1"/s/\[fw1, shaper1\]/[fw2, shaper1]/' vnffg.yaml
python3 nfvo.py apply vnffg.yaml
until docker exec clab-$STUDENT-l6-client1 ping -c 1 -W 1 10.$N.6.100 >/dev/null 2>&1; do sleep 0.2; done; date +%T.%N
docker exec clab-$STUDENT-l6-client1 ping -c 2 10.$N.6.100
sudo env STUDENT=$STUDENT N=$N ./vnfm.sh terminate fw1 3 4
./vnfm.sh list
```

Хронология (25.11.2026):

| Время | Действие | Результат |
| --- | --- | --- |
| 13:20:24 | instantiate fw2 | порты OVS 5 и 6 |
| — | c2 → fw2, apply | трафик client2 через fw2 |
| 13:24:08 | `docker stop` fw1 | client1: 100 % потерь |
| 13:24:13 | c1 → fw2, apply | ping client1 восстановлен |
| — | `terminate fw1 3 4` | остались fw2 и shaper1 |

Время восстановления около 4,7 с.

### 6. Производительность

```bash
docker exec clab-$STUDENT-l6-client1 iperf3 -c 10.$N.6.100 -t 10 -R -P 4 > /tmp/iperf.txt &
sleep 3
docker exec clab-$STUDENT-l6-sw sh -c "top -b -n 2 -d 2 | awk '/^top -/{n++} n==2' | head -15"
wait
tail -8 /tmp/iperf.txt
```

| Тест | Результат |
| --- | --- |
| client1 → server | 11,9 Мбит/с (шейпер) |
| server → client1 (`-R`) | 936 Мбит/с |
| `-R -P 4` | 1,62 Гбит/с, `ovs-vswitchd` 100 % одного ядра |

### 7. Проверка блокировки на fw2 и итог

```bash
docker exec clab-$STUDENT-l6-fw2 nft list ruleset | grep block
docker exec clab-$STUDENT-l6-client1 curl -s -m 3 -o /dev/null -w 'client1 :8082 HTTP %{http_code}\n' http://10.$N.6.100:8082/
docker exec clab-$STUDENT-l6-client2 curl -s -m 3 -o /dev/null -w 'client2 :8082 HTTP %{http_code}\n' http://10.$N.6.100:8082/
docker exec clab-$STUDENT-l6-fw2 nft list ruleset | grep block
./vnfm.sh list
python3 nfvo.py show vnffg.yaml | head -12
docker exec clab-$STUDENT-l6-fw2 nft list ruleset | grep counter
docker exec clab-$STUDENT-l6-client1 iperf3 -c 10.$N.6.100 -t 3 | tail -3
```

### 8. Остановка стенда

```bash
sudo env STUDENT=$STUDENT N=$N containerlab destroy -t lab6.clab.yml --cleanup
docker rm -f clab-$STUDENT-l6-fw2
```
