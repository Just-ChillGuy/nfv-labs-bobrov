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
- Образы `lab-tools:1.0`, `ovs-lab:latest`, `odl:chromium`
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

### 1. Развёртывание и контроллер

```bash
export STUDENT=bobrov N=82
sudo env STUDENT=$STUDENT N=$N containerlab deploy -t lab6.clab.yml
docker exec -d clab-bobrov-l6-server iperf3 -s
docker exec -d clab-bobrov-l6-server python3 -m http.server 8082
docker exec -t clab-bobrov-l6-odl /opt/odl/bin/client "feature:install odl-openflowplugin-flow-services-rest odl-openflowplugin-app-table-miss-enforcer"
docker exec -i -e ODL_IP=10.106.82.100 clab-bobrov-l6-sw sh < scripts/sw-setup.sh
```

До применения цепочки ping не проходит: OVS в режиме `fail-mode=secure` работает только по потокам.

### 2. Настройка VNF

```bash
./vnfm.sh configure fw1 fw
./vnfm.sh configure shaper1 shaper
```

### 3. Сервисная цепочка

```bash
python3 nfvo.py apply vnffg.yaml
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

HTTP проходит (200), порт 8082 блокируется (000).

### 4. Scale-out

```bash
sudo env STUDENT=$STUDENT N=$N ./vnfm.sh instantiate fw2 fw 5 6
```

Цепочка c2 переведена на `[fw2, shaper1]`, счётчик `allow http` растёт только на fw2.

### 5. Heal и terminate

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

| Тест | Результат |
| --- | --- |
| client1 → server | 11,9 Мбит/с (шейпер) |
| server → client1 (`-R`) | 936 Мбит/с |
| `-R -P 4` | 1,62 Гбит/с, `ovs-vswitchd` 100 % одного ядра |

