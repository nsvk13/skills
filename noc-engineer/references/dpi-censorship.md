# DPI, цензура и обход блокировок — Россия 2024-2025

## Философия подхода

Этот файл — не просто справочник. Когда пользователь описывает проблему с блокировкой:
1. **Диагностируй сначала** — определи точный метод блокировки (IP, SNI, протокол, whitelist)
2. **Учитывай контекст** — регион, оператор, тип подключения (мобильный/домашний) дают разные ограничения
3. **Предлагай решение под ситуацию** — не один вариант, а оптимальный с объяснением почему
4. **Думай на опережение** — что будет делать регулятор дальше, как сделать решение устойчивым
5. **Проверяй актуальность** — репо и ситуация меняются, смотри свежие issues/коммиты

---

## Модели блокировок в РФ

### 1. Blacklist (обычные блокировки)
Реестр РКН + ТСПУ на стыках операторов. Блокируется конкретный домен/IP.

**Признаки:** работает из-за рубежа, не работает в РФ. traceroute обрывается на узлах оператора.

**Решение:** любой VPN/прокси с зарубежным сервером.

### 2. Whitelist (мобильный интернет под ограничениями)
Разрешён только ограниченный список ресурсов. **Принципиально другая модель** — обычный VPN не поможет, если его сервер не в whitelist.

**Признаки:**
- Мобильный интернет, работает только часть сайтов
- `connection reset` или `timeout` на нелистованные ресурсы
- Даже ping до зарубежных IP не проходит
- Варьируется по оператору и даже по вышке

**Регионы:** преимущественно Северный Кавказ, зоны боевых действий, иногда вводится временно по всей РФ при обострениях.

**Решение:** нужен РФ-сервер с "белым" IP (из разрешённых диапазонов) как входная точка.

### 3. Throttling без полной блокировки
Трафик до ресурса замедляется до 14 кбит/с (или хуже). Пример: YouTube 2023-2024.

**Признаки:** сайт открывается, но очень медленно. traceroute доходит, но latency высокая.

**Решение:** zapret/zapret2/b4 (DPI-десинхронизация) — не нужен внешний сервер.

### 4. Протокольная блокировка
ТСПУ детектирует VPN-протоколы по сигнатурам и блокирует.

**Признаки:** VPN подключается, но трафик не идёт или соединение рвётся через несколько секунд.

**Решение:** обфускация (AmneziaWG, VLESS+Reality, Hysteria2).

---

## Диагностика: как определить тип блокировки

```bash
# 1. Базовая проверка
ping 8.8.8.8                          # ICMP до Google DNS
curl -v https://example.com           # HTTP(S) до заблокированного сайта
curl -v --connect-timeout 5 https://IP_адрес  # напрямую по IP (минуя DNS)

# 2. Определить на каком уровне блокировка
dig +short blocked-site.com           # DNS отвечает? Правильный IP?
dig @8.8.8.8 blocked-site.com        # Через Google DNS — отличается?
curl -v --resolve blocked-site.com:443:$(dig +short blocked-site.com) https://blocked-site.com
# vs
curl -v --resolve blocked-site.com:443:<другой_IP> https://blocked-site.com
# Если второй работает — SNI блокировка, а не IP

# 3. traceroute — где обрывается
mtr -n --report blocked-site.com
# Обрыв на AS8359 (МТС), AS8976 (Ростелеком), AS3216 (Билайн) → ТСПУ

# 4. Проверить whitelist режим
# С мобильного — пингуй разные IP:
ping 8.8.8.8        # Google — скорее всего заблокирован
ping 77.88.8.8      # Яндекс DNS — в whitelist
ping 51.250.x.x     # Yandex Cloud — в whitelist
# Если российские отвечают, а зарубежные нет → whitelist режим
```

---

## Инструменты: обзор и когда использовать

### zapret / zapret2 (bol-van)
- **GitHub:** https://github.com/bol-van/zapret (основной), https://github.com/bol-van/zapret2 (новая версия)
- **Что делает:** манипуляция пакетами через NFQUEUE/WFP для десинхронизации DPI
- **Когда:** throttling, SNI-блокировки, без нужды во внешнем сервере
- **Платформы:** Linux, Windows, OpenWrt, Keenetic
- **Не поможет:** whitelist режим (трафик физически не проходит)

```bash
# Автоопределение метода для провайдера
cd zapret && ./blockcheck.sh

# Типовой запуск nfqws против YouTube throttling
nfqws --daemon \
  --qnum=200 \
  --dpi-desync=fake,disorder2 \
  --dpi-desync-ttl=5 \
  --dpi-desync-fooling=md5sig \
  --hostlist=/opt/zapret/ipset/zapret-hosts-user.txt

# iptables перехват
iptables -t mangle -A OUTPUT -p tcp --dport 443 \
  -m connbytes --connbytes 1:6 --connbytes-dir=original --connbytes-mode=segments \
  -m mark ! --mark 0x40000000/0x40000000 \
  -j NFQUEUE --queue-num 200 --queue-bypass
```

**zapret2** — переработанная версия с улучшенной архитектурой, blockcheck2 скриптом. Актуальнее для новых установок.

### b4 (DanielLavrushin)
- **GitHub:** https://github.com/DanielLavrushin/b4
- **Что делает:** то же что zapret, но с GUI и проще в настройке
- **Когда:** Windows, нужен интерфейс, не хочется возиться с командной строкой
- **1.1k звёзд**, активно поддерживается

### Zapret + AmneziaVPN на роутере (Zzoomrus)
- **GitHub:** https://github.com/Zzoomrus/Zapret-AmneziaVPN
- **Что делает:** комбо — zapret для обхода замедления YouTube + AmneziaWG для остального
- **Когда:** роутер (Keenetic и другие), хочется чтобы работало для всех устройств в сети
- Поддерживает Keenetic через отдельный README

### MTProxy (Telegram)
- **GitHub:** https://github.com/TelegramMessenger/MTProxy
- **Что делает:** прокси для Telegram по протоколу MTProto
- **Когда:** Telegram заблокирован, нужен только он
- **Порт:** рекомендуется 8443 (443 часто занят)
- Разворачивается рядом с Amnezia-контейнерами без конфликтов

```bash
docker run -d \
  --name mtproxy \
  -p 8443:443 \
  -e SECRET=$(openssl rand -hex 16) \
  telegrammessenger/proxy:latest
```

### whitebox (quyxishi) — мониторинг AmneziaWG
- **GitHub:** https://github.com/quyxishi/whitebox
- **Что делает:** Prometheus exporter для AmneziaWG (форк с поддержкой amnezia-xray-core)
- **Когда:** нужен мониторинг AWG нод в Grafana
- Заменяет зависимость xray-core на amnezia-xray-core

### xraycheck (WhitePrime)
- **GitHub:** https://github.com/WhitePrime/xraycheck
- **Что делает:** технический анализ конфигов VLESS/VMess/Trojan/SS/Hysteria/MTProto с эмуляцией сетевых ограничений
- **Когда:** нужно проверить/отладить конфиг VPN без реального ограниченного подключения; тест на устойчивость к DPI
- Полезен для автоматизированного тестирования конфигов

### RealiTLScanner (XTLS)
- **GitHub:** https://github.com/XTLS/RealiTLScanner
- **Что делает:** сканирует диапазон IP, ищет серверы подходящие для Reality (TLS 1.3, h2, нужный SNI)
- **Когда:** выбор оптимального `dest` домена для Reality конфига

```bash
# Скомпилировать и запустить
go build -o RealiTLScanner .
./RealiTLScanner -ip 1.1.1.0/24 -timeout 3 -thread 100
# Выдаёт список IP с доменами, поддерживающими TLS 1.3
```

---

## Whitelist bypass: детальный разбор

### Диапазоны IP в whitelist (актуально начало 2025)

| CIDR | Провайдер | Операторы | Примечания |
|------|-----------|-----------|------------|
| `51.250.0.0/17` | Yandex Cloud | МТС, Мегафон, Билайн, Т2, Yota | Самый надёжный вариант |
| `84.201.0.0/16` | Yandex Cloud | МТС, Мегафон, Т2, Билайн | Широкое покрытие |
| `95.163.248.0/22` | VK Cloud (MSM) | Все основные | Часто занят, сложно получить |
| `217.16.24.0/21` | VK Cloud | МТС, Т2 | |
| `158.160.0.0/17` | Yandex Cloud | Т2 | Снижается актуальность |
| `185.39.206.0/24` | TimeWeb Cloud | Т2 | |
| `91.222.239.0/24` | TimeWeb Cloud | Билайн, Т-Моб | |

**Актуальный список:** https://github.com/hxehex/russia-mobile-internet-whitelist — обновляется, содержит `cidrwhitelist.txt` и `whitelist.txt` (домены).

**Важно:** ситуация хаотична. Yota (дочка Мегафона) может иметь более строгий whitelist чем Мегафон на той же вышке. В деревнях бывает полный blackout. Варьируется до уровня конкретной вышки.

### Получение нужного IP в Yandex Cloud

```bash
# Диапазон 51.250.x.x:
# При создании ВМ включить "Защита от DDoS" → зона a или b
# ~95% случаев выдаётся 51.250.x.x

# Диапазон 84.201.x.x:
# Без DDoS защиты, зона a/b/d

# Если выпал не тот диапазон — пересоздать ВМ (несколько попыток)
# Проверить ДО настройки:
ping <new-vm-ip>  # с мобильного в зоне ограничений
```

### Архитектуры обхода whitelist

**Вариант 1: Двойной Xray (3x-ui на обоих серверах)**
```
Клиент → РФ сервер (белый IP, 3x-ui) → зарубежный сервер (3x-ui) → интернет
```
Плюс: удобное управление через UI, легко добавлять клиентов
Минус: два слоя терминации TLS, чуть больше latency

**Вариант 2: nginx stream / iptables DNAT (проще)**
```
Клиент → РФ сервер (белый IP, nginx/iptables) → зарубежный сервер → интернет
```
```nginx
# /etc/nginx/nginx.conf
stream {
    server {
        listen 443;
        proxy_pass <зарубежный>:443;
        proxy_buffer_size 16k;
    }
}
```
```bash
# Или через iptables
iptables -t nat -A PREROUTING -p tcp --dport 443 \
  -j DNAT --to-destination <зарубежный>:443
iptables -t nat -A POSTROUTING -j MASQUERADE
sysctl -w net.ipv4.ip_forward=1
```
Плюс: нет двойной TLS терминации, меньше latency, меньше точек отказа
Минус: меньше контроля над клиентами

**Вариант 3: VK TURN proxy (vk-turn-proxy / whitelist-bypass)**
```
Клиент → VK TURN серверы (155.212.x.x:19302, в whitelist) → зарубежный сервер
```
- **vk-turn-proxy** (cacggghp): туннелирует WireGuard через TURN серверы VK звонков по STUN ChannelData
- **whitelist-bypass** (kulikov0): туннелирует весь трафик через WebRTC DataChannel VK звонка
- Логин/пароль для TURN генерируются из ссылки на VK звонок
- Трафик выглядит как обычный VK звонок

```bash
# vk-turn-proxy — сервер
./server -listen 0.0.0.0:56000 -connect 127.0.0.1:<wg-port>

# Клиент (Linux)
./client-linux -peer <server-ip>:56000 -link <vk-call-link> -listen 127.0.0.1:9000 | sudo bash routes.sh
# В WireGuard конфиге: Endpoint = 127.0.0.1:9000, MTU = 1420
```

**SNI для Reality в whitelist условиях**
Домен `dest` должен быть доступен через белый IP. Проверенные варианты:
- `ads.x5.ru` — работает на большинстве операторов во многих регионах
- `gosuslugi.ru`, `www.sberbank.ru` — как правило в whitelist везде
- Для подбора: RealiTLScanner по диапазону `51.250.0.0/17`

---

## Списки доменов и IP для маршрутизации

### itdoginfo/allow-domains
- **GitHub:** https://github.com/itdoginfo/allow-domains
- **Что это:** актуальный список заблокированных доменов в РФ в множестве форматов
- **Форматы:** dnsmasq nfset/ipset, sing-box, xray geosite.dat, ClashX, Mikrotik, Kvas, RAW
- **Категории:** YouTube, Discord, Meta, TikTok, Twitter, Anime, GeoBlock, Porn, News + H.O.D.C.A (Hetzner/OVH/DO/CF/AWS)

```bash
# Xray geosite.dat
curl -L https://github.com/itdoginfo/allow-domains/releases/latest/download/geosite.dat \
  -o /usr/local/etc/xray/geosite.dat

# RAW список (Россия inside)
curl https://raw.githubusercontent.com/itdoginfo/allow-domains/main/Russia/inside-raw.lst

# dnsmasq nfset (для OpenWrt >=23.05)
curl https://raw.githubusercontent.com/itdoginfo/allow-domains/main/Russia/inside-dnsmasq-nfset.lst
```

Логика: добавляем resolved IP заблокированных доменов в nftables set `vpn_domains`, потом маршрутизируем через VPN только их.

### hxehex/russia-mobile-internet-whitelist
- **GitHub:** https://github.com/hxehex/russia-mobile-internet-whitelist
- **Что это:** список доменов и IP которые остаются доступными в whitelist режиме
- **Файлы:** `whitelist.txt` (домены), `cidrwhitelist.txt` (CIDR), `ipwhitelist.txt`
- Полезен для понимания что можно использовать как "легальный" транспорт

---

## Аналитический фреймворк: как выбрать решение

### Decision tree

```
Проблема с интернетом
├── Мобильный интернет, большинство сайтов недоступно?
│   └── WHITELIST режим
│       ├── Есть РФ сервер с белым IP?
│       │   ├── Да → nginx/iptables forward + VLESS+Reality на зарубежном
│       │   └── Нет → VK TURN proxy (не нужен свой сервер) или получить YC VM
│       └── Нет доступа к настройке сервера?
│           └── VK TURN proxy (whitelist-bypass или vk-turn-proxy)
│
├── Конкретные сайты заблокированы (обычная блокировка)?
│   ├── Нужно только для этих сайтов (не весь трафик)?
│   │   └── zapret/zapret2/b4 — без внешнего сервера
│   └── Нужен полный тоннель?
│       ├── UDP не заблокирован → AmneziaWG 1.5 или Hysteria2
│       └── UDP заблокирован → VLESS + Reality (TCP 443)
│
├── VPN подключается но рвётся/медленный?
│   ├── Стандартный WireGuard → замени на AmneziaWG
│   ├── Подозрение на DPI → xraycheck для диагностики
│   └── MTU проблемы → ping -M do -s 1400 <host>, уменьшай MTU
│
└── YouTube/конкретный сервис тормозит?
    └── zapret/zapret2 с blockcheck.sh — определит метод автоматически
```

### Устойчивость решений к эскалации

| Решение | Устойчивость | Что может сломать |
|---------|-------------|-------------------|
| VLESS + Reality | Очень высокая | Ничего из известного на 2025 |
| AmneziaWG 1.5 | Высокая | Если РКН начнёт блокировать UDP массово |
| Hysteria2 | Высокая | Блокировка QUIC/UDP у провайдера |
| zapret/zapret2 | Средняя | Обновление сигнатур ТСПУ |
| VK TURN proxy | Средняя | Если VK заблокирует кастомные DataChannel |
| nginx forward через YC IP | Средняя | YC бан аккаунта, изменение whitelist |

### Комбинированные подходы (наиболее устойчиво)

**Для мобильного whitelist:**
```
AmneziaWG 1.5 на РФ сервере (YC 51.250.x.x) → пробрасывает на зарубежный сервер
```
Лучше чем двойной VLESS: AWG быстрее, меньше overhead.

**Для домашнего на роутере:**
```
zapret (YouTube + заблокированные сайты без VPN) 
+ AmneziaWG (всё остальное, selectively)
```
Реализовано в Zzoomrus/Zapret-AmneziaVPN для Keenetic.

**Максимальная устойчивость:**
```
VLESS + Reality (основной) + Hysteria2 (резерв при TCP блокировке)
+ список доменов через itdoginfo/allow-domains для split-tunneling
```

---

## Мониторинг и диагностика инфраструктуры

### whitebox — Prometheus exporter для AWG
```yaml
# docker-compose.yml
services:
  whitebox:
    image: ghcr.io/quyxishi/whitebox:latest
    network_mode: host
    cap_add: [NET_ADMIN]
    volumes:
      - /etc/amnezia/awg:/etc/amnezia/awg:ro
```

Метрики в Grafana: количество пиров, трафик per-peer, last handshake.

### xraycheck — тестирование конфигов
```bash
# Установка
git clone https://github.com/WhitePrime/xraycheck
cd xraycheck && pip install -r requirements.txt

# Проверить конфиг с эмуляцией ограничений
python xraycheck.py --config vless-reality.json --simulate-ru
```

### Автоматизированная проверка доступности
```bash
# Скрипт проверки белого IP
#!/usr/bin/env bash
check_whitelist() {
  local ip=$1
  # Пробуем подключиться к серверу через известно-заблокированный IP
  timeout 3 nc -zv $ip 443 2>/dev/null && echo "OK: $ip" || echo "BLOCKED: $ip"
}

# Список тестовых IP (заведомо заблокированных)
for ip in 8.8.8.8 1.1.1.1 142.250.0.0; do
  check_whitelist $ip
done
```

---

## Актуальные ссылки для мониторинга ситуации

```
# Обновляй при каждом обращении к этой теме:
https://github.com/hxehex/russia-mobile-internet-whitelist  — актуальный whitelist
https://github.com/itdoginfo/allow-domains                   — актуальный blocklist
https://github.com/bol-van/zapret2                           — свежие методы DPI обхода
https://github.com/apernet/hysteria                          — обновления Hysteria2
https://github.com/XTLS/RealiTLScanner                       — Reality SNI сканер
```

**Что смотреть в репо:** последние коммиты, открытые issues (часто там обсуждают что перестало работать), README изменения.

---

## Риски и ограничения

| Риск | Вероятность | Митигация |
|------|-------------|-----------|
| Бан аккаунта Yandex Cloud | Средняя | Регистрировать на разные аккаунты, не превышать разумный трафик |
| Изменение whitelist диапазонов | Высокая (регулярно) | Мониторить hxehex/russia-mobile-internet-whitelist |
| Блокировка VK TURN серверов | Низкая (VK сам в whitelist) | — |
| Детектирование Reality | Очень низкая | Регулярно обновлять Xray-core |
| Блокировка UDP у провайдера | Средняя для мобильных | Иметь TCP-fallback (Reality) |