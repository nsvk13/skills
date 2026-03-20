# VPN Protocols Reference

## Ландшафт протоколов (2024-2025)

| Протокол | Обнаруживаемость DPI | Скорость | Сложность | Статус |
|----------|---------------------|----------|-----------|--------|
| WireGuard (стандарт) | Высокая — легко детектируется | ★★★★★ | Низкая | Заблокирован в РФ |
| AmneziaWG 1.5 | Низкая — рандомизация handshake | ★★★★★ | Низкая | Активно используется |
| VLESS + Reality | Очень низкая — мимикрия под HTTPS | ★★★★ | Средняя | Основной выбор 2024 |
| Hysteria2 | Низкая — UDP/QUIC обфускация | ★★★★★ | Средняя | Отлично для высоких latency |
| Shadowsocks (AEAD) | Средняя — детектируется по энтропии | ★★★★ | Низкая | Устаревает |
| SS + v2ray-plugin | Низкая | ★★★ | Средняя | Рабочий вариант |
| Trojan | Низкая — TLS мимикрия | ★★★★ | Средняя | Стабильный |
| OpenVPN | Очень высокая | ★★★ | Средняя | Заблокирован |
| IPSec/L2TP | Очень высокая | ★★★ | Высокая | Заблокирован |

---

## AmneziaWG 1.5

Форк WireGuard с обфускацией handshake и трафика. Основные изменения vs стандартный WG:

**Параметры обфускации** (в конфиге клиента/сервера):
```ini
[Interface]
# ...стандартные поля WireGuard...
Jc = 4          # количество junk пакетов в handshake initiation
Jmin = 40       # минимальный размер junk пакета (байт)
Jmax = 70       # максимальный размер junk пакета
S1 = 0          # добавочный размер к первому handshake пакету
S2 = 0          # добавочный размер ко второму handshake пакету
H1 = 1831716540 # рандомные magic header значения (uint32)
H2 = 2048348640
H3 = 1297918304
H4 = 3444412847
```

**Рекомендуемые значения против ТСПУ:**
```ini
Jc = 4
Jmin = 40
Jmax = 70
S1 = 0
S2 = 0
# H1-H4 — генерируй случайно, не используй дефолты
```

**Установка сервера:**
```bash
# Docker (рекомендуется)
docker run -d \
  --name amnezia-wg \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  -p 51820:51820/udp \
  -v /opt/amnezia/awg:/etc/wireguard \
  --sysctl net.ipv4.ip_forward=1 \
  amneziavpn/amnezia-wg

# Или нативно через amnezia-wg пакет
apt install amnezia-wg
```

**Важно:** клиент и сервер должны использовать одинаковые H1-H4 значения. Клиенты: официальное приложение Amnezia, awg-quick.

---

## VLESS + Reality

Протокол из экосистемы Xray-core. Reality — замена TLS, использует реальный сертификат чужого сайта (SNI-прокси).

**Принцип работы:**
1. Клиент делает TLS handshake с реальным доменом (например `microsoft.com`)
2. Сервер с Reality различает своих клиентов по shortId
3. Чужие клиенты (DPI, сканеры) получают реальный трафик microsoft.com — не отличить

**Конфиг сервера (xray/config.json):**
```json
{
  "inbounds": [{
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{
        "id": "<uuid>",
        "flow": "xtls-rprx-vision"
      }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "dest": "microsoft.com:443",
        "serverNames": ["microsoft.com", "www.microsoft.com"],
        "privateKey": "<private-key>",
        "shortIds": ["<8-byte-hex>"]
      }
    }
  }]
}
```

**Генерация ключей:**
```bash
xray x25519                    # генерирует пару private/public key
openssl rand -hex 8            # shortId
cat /proc/sys/kernel/random/uuid  # UUID для клиента
```

**Выбор SNI домена — критично:**
- Домен должен поддерживать TLS 1.3
- Сервер должен отвечать на том же IP что и домен (или близко)
- Популярные: `microsoft.com`, `apple.com`, `dl.google.com`
- Проверить: `curl -v --resolve microsoft.com:443:<your-server-ip> https://microsoft.com`

**Клиенты:** v2rayN (Windows), v2rayNG (Android), Nekoray, Hiddify, Streisand

---

## Hysteria2

Протокол поверх QUIC (UDP) с агрессивным контролем перегрузки. Отлично работает на плохих каналах и при высоком latency.

**Конфиг сервера (`/etc/hysteria/config.yaml`):**
```yaml
listen: :443

tls:
  cert: /etc/letsencrypt/live/domain.com/fullchain.pem
  key:  /etc/letsencrypt/live/domain.com/privkey.pem

auth:
  type: password
  password: "strongpassword"

masquerade:
  type: proxy
  proxy:
    url: https://news.ycombinator.com  # реальный сайт для маскировки
    rewriteHost: true

# Опционально — ограничение скорости
bandwidth:
  up: 100 mbps
  down: 1 gbps
```

**Конфиг клиента:**
```yaml
server: domain.com:443
auth: strongpassword

bandwidth:
  up: 50 mbps
  down: 200 mbps

tls:
  sni: domain.com

socks5:
  listen: 127.0.0.1:1080
http:
  listen: 127.0.0.1:8080
```

**Запуск:**
```bash
# Сервер
hysteria-linux-amd64 server --config /etc/hysteria/config.yaml

# Клиент
hysteria-linux-amd64 client --config client.yaml
```

**Проблемы с UDP блокировкой:**
Некоторые провайдеры блокируют UDP полностью или ограничивают QUIC. Проверить:
```bash
# С клиентской машины
nc -u <server> 443
# Или
nmap -sU -p 443 <server>
```
Если UDP заблокирован → используй VLESS+Reality (TCP).

---

## Shadowsocks + обфускация

Базовый SS детектируется по статистическому анализу (высокая энтропия, характерный паттерн). Нужна обфускация.

**SS + obfs4 (через lyrebird):**
```bash
# Сервер
ss-server -c /etc/shadowsocks/config.json \
  --plugin obfs-server \
  --plugin-opts "obfs=tls;obfs-host=www.bing.com"

# Конфиг
{
  "server": "0.0.0.0",
  "server_port": 443,
  "password": "password",
  "method": "chacha20-ietf-poly1305",
  "plugin": "obfs-server",
  "plugin_opts": "obfs=tls"
}
```

**Рекомендуемые шифры (2024):**
- `chacha20-ietf-poly1305` — быстрый, безопасный
- `aes-256-gcm` — если есть AES-NI на CPU
- Старые (`aes-256-cfb`, `rc4-md5`) — не использовать

---

## Сравнение для РФ (2024-2025)

**Лучший выбор по сценарию:**

| Сценарий | Протокол |
|----------|----------|
| Максимальная надёжность | VLESS + Reality |
| Быстрый мобильный | AmneziaWG 1.5 |
| Плохой канал / высокий latency | Hysteria2 |
| Простая самостоятельная настройка | AmneziaWG 1.5 |
| Массовая раздача клиентам | AmneziaWG или Hysteria2 |
| UDP заблокирован у провайдера | VLESS + Reality |