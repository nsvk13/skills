# DPI и цензура — обход блокировок

## Как работает ТСПУ (Россия)

ТСПУ (Технические Средства Противодействия Угрозам) — система глубокой инспекции пакетов, установленная на стыках операторов связи согласно Постановлению Правительства №1316/1351.

**Архитектура:**
```
Пользователь → Оператор связи → [ТСПУ оборудование] → Интернет
                                        ↑
                              Эшелон (Ростелеком)
                              реализует блокировки
                              по команде РКН
```

**Что умеет ТСПУ:**
- Блокировка по IP (реестр РКН)
- Блокировка по SNI (TLS ClientHello)
- DPI по сигнатурам протоколов (WireGuard handshake, OpenVPN, etc.)
- Статистический анализ — высокая энтропия → подозрительно
- Замедление (throttling) без полной блокировки — Twitter 2021
- BGP-манипуляции для отдельных AS

**Что НЕ умеет (пока):**
- Расшифровывать TLS 1.3 с правильным SNI
- Отличить Reality от настоящего HTTPS без shortId
- Блокировать QUIC без ущерба для легального трафика

---

## Методы блокировок

### 1. IP-блокировка
Простейший метод. РКН ведёт реестр IP.

Обход: смена IP сервера, IPv6, CDN (Cloudflare).

### 2. SNI-фильтрация
DPI читает поле `server_name` в TLS ClientHello (до шифрования).

```
TLS ClientHello → SNI: "blocked-site.com" → блокировать
```

Обходы:
- **ESNI/ECH** (Encrypted Client Hello) — SNI зашифрован, но поддержка пока ограничена
- **Reality** — SNI ведёт на легальный сайт
- **Domain Fronting** — SNI ≠ Host header (CDN)

### 3. Fingerprinting протоколов
DPI знает сигнатуры WireGuard, OpenVPN, etc.

**WireGuard handshake initiation** (первые 4 байта = `0x01000000`) — легко детектируется.
AmneziaWG рандомизирует эти байты.

### 4. Статистический анализ
Трафик с высокой энтропией без распознаваемого протокола → подозрительно.
Решение: мимикрия под TLS/HTTPS (Reality, Trojan, obfs4 с `obfs=tls`).

### 5. Timing analysis
Анализ паттернов соединений. Против него сложнее — нужен traffic padding.

---

## zapret — пассивный обход для браузера

[zapret](https://github.com/bol-van/zapret) — инструмент для обхода DPI без VPN, на уровне TCP/UDP манипуляций.

**Принцип:** модифицирует пакеты так, чтобы DPI не смог правильно проанализировать трафик, но сервер всё равно принял соединение.

**Методы:**
- `--dpi-desync` — десинхронизация TCP потока для DPI
- `--hostlist` — применять только к заблокированным доменам
- Фрагментация TLS ClientHello
- TTL манипуляции

**Установка (Linux):**
```bash
git clone https://github.com/bol-van/zapret
cd zapret
./install_easy.sh   # интерактивный установщик

# Или вручную
./blockcheck.sh     # автоопределение метода для твоего провайдера
```

**nfqws — основной компонент:**
```bash
# Пример запуска
nfqws --daemon \
  --qnum=200 \
  --dpi-desync=fake,disorder2 \
  --dpi-desync-ttl=5 \
  --dpi-desync-fooling=md5sig \
  --hostlist=/opt/zapret/ipset/zapret-hosts-user.txt

# iptables правила для перехвата
iptables -t mangle -A OUTPUT -p tcp --dport 443 \
  -m connbytes --connbytes 1:6 --connbytes-dir=original --connbytes-mode=segments \
  -m mark ! --mark 0x40000000/0x40000000 \
  -j NFQUEUE --queue-num 200 --queue-bypass
```

**tpws — для HTTP:**
```bash
tpws --daemon --port=988 --split-pos=2 --hostlist=/opt/zapret/ipset/...
```

**Определение рабочего метода:**
```bash
cd zapret && ./blockcheck.sh
# Скрипт автоматически тестирует разные методы и показывает что работает
```

---

## Проверка блокировок

### Проверить заблокирован ли ресурс
```bash
# Через разные DNS резолверы
dig @8.8.8.8 blocked-site.com
dig @1.1.1.1 blocked-site.com
dig @77.88.8.8 blocked-site.com  # Яндекс DNS

# Сравнить с локальным резолвером
dig blocked-site.com

# Traceroute — где обрывается?
mtr --report --no-dns blocked-site.com
traceroute -n blocked-site.com

# Проверить SNI блокировку
curl -v --resolve blocked-site.com:443:<другой-ip> https://blocked-site.com
```

### Определить метод блокировки
```bash
# 1. Получить IP
dig +short blocked-site.com

# 2. Проверить IP напрямую
curl -v https://<ip> -H "Host: blocked-site.com"

# 3. Проверить с другим SNI
curl -v --insecure https://<ip> --resolve fake-domain.com:443:<ip>

# Если (2) работает а через домен нет → SNI блокировка
# Если IP тоже не работает → IP блокировка
```

### Реестр РКН
```bash
# Проверить есть ли IP/домен в реестре
# https://eais.rkn.gov.ru/ — официальный
# https://blocklist.rkn.gov.ru/ — API
curl "https://blocklist.rkn.gov.ru/check/?domain=example.com"
```

---

## DNS — обход и DoH/DoT

### DNS over HTTPS (DoH)
```bash
# Проверить через DoH вручную
curl -s "https://cloudflare-dns.com/dns-query?name=blocked-site.com&type=A" \
  -H "accept: application/dns-json"

curl -s "https://dns.google/resolve?name=blocked-site.com&type=A"
```

### Настройка DoH в systemd-resolved
```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google
DNSOverTLS=yes
DNSSEC=yes
```

### AdGuard DNS / NextDNS как обход
Для мобильных — встроенный "Private DNS" в Android: `dns.adguard.com`

---

## Белые списки (whitelist-режим) — отдельная модель блокировок

В отличие от обычных блокировок (blacklist), белые списки — **разрешён только ограниченный набор ресурсов**, всё остальное дропается. Применяется в отдельных регионах РФ (преимущественно Северный Кавказ, Донецк/Луганск зоны), периодически вводится при обострениях.

### Принцип обхода

Нужен **РФ сервер с "белым" IP** (из диапазонов, разрешённых операторами) как точка входа:

```
Клиент → РФ сервер (белый IP) → Зарубежный сервер → Интернет
```

РФ сервер либо:
- Проксирует трафик через 3x-ui + Xray (два инстанса VLESS+Reality)
- Или просто форвардит порт 443 через nginx stream / iptables (проще, меньше latency)

### Диапазоны IP в белых списках операторов (актуально на начало 2025)

| CIDR | Провайдер | Операторы |
|------|-----------|-----------|
| `51.250.0.0/17` | Yandex Cloud | Билайн, Т-Моб, МТС, Yota, Мегафон, Т2 |
| `84.201.0.0/16` | Yandex Cloud | МТС, Мегафон, Т2, Билайн |
| `158.160.0.0/17` | Yandex Cloud | Т2 (актуальность снижается) |
| `95.163.248.0/22` | VK Cloud (Digital Networks MSM) | Билайн, Т-Моб, МТС, Yota, Мегафон, Т2 |
| `217.16.24.0/21` | VK Cloud | МТС, Т2 |
| `185.39.206.0/24` | TimeWeb Cloud | Т2 |
| `91.222.239.0/24` | TimeWeb Cloud | Билайн, Т-Моб |

> Диапазоны могут меняться — операторы добавляют/убирают. Актуальный список проверяй в тематических чатах.

**Как получить IP из нужного диапазона (Yandex Cloud):**
- Диапазон `51.250.x.x` — при создании ВМ включить защиту от DDoS, выбрать зону `a` или `b`; в ~95% случаев выдаётся `51.250.x.x`
- Если выпал другой диапазон — пересоздать ВМ
- Диапазон `84.201.x.x` — без DDoS защиты, тоже рабочий на большинстве операторов

**Проверка IP перед настройкой:**
```bash
# С мобильного интернета в зоне с белыми списками
ping <ip-сервера>
# Если отвечает — IP в белом списке, можно использовать
```

### Схема через nginx stream (проще чем двойной Xray)

```nginx
# /etc/nginx/nginx.conf
stream {
    server {
        listen 443;
        proxy_pass <зарубежный-сервер>:443;
        proxy_buffer_size 16k;
    }
}
```

```bash
# Или через iptables (ещё проще)
iptables -t nat -A PREROUTING -p tcp --dport 443 \
  -j DNAT --to-destination <зарубежный-сервер>:443
iptables -t nat -A POSTROUTING -j MASQUERADE
```

Плюс nginx/iptables подхода: не нужно терминировать TLS дважды, меньше latency, меньше точек отказа.

### SNI для Reality в условиях белых списков

Домен для Reality (`dest` / `serverNames`) должен быть доступен **через белый IP** — иначе handshake не пройдёт. Проверенные варианты:
- `ads.x5.ru` — работает на всех основных операторах в большинстве регионов
- `www.sberbank.ru`, `gosuslugi.ru` — как правило в белых списках везде

### Риски

- Яндекс и VK Cloud могут заблокировать аккаунт при обнаружении VPN-трафика (прецеденты есть)
- РКН постепенно сужает диапазоны — то что работает сегодня, может не работать через месяц
- При активном использовании IP может попасть в блок

---

## Полезные ресурсы и инструменты

```bash
# Проверка связности из разных точек мира
# https://check-host.net
# https://ping.pe

# Анализ BGP / AS
whois -h whois.radb.net <ip>
curl "https://stat.ripe.net/data/prefix-overview/data.json?resource=<ip>"

# Что за AS у IP
curl "https://ipinfo.io/<ip>"

# Сканирование открытых портов (для своего сервера)
nmap -sS -p- --min-rate 1000 <server-ip>
nmap -sU -p 51820,443,8443 <server-ip>   # UDP порты
```