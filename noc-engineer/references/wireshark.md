# Wireshark & Анализ трафика

## Базовые фильтры (display filters)

### По протоколу
```
tcp
udp
dns
http
tls
quic
icmp
arp
wireguard
```

### По адресу / порту
```
ip.addr == 192.168.1.1
ip.src == 192.168.1.1
ip.dst == 10.0.0.1
tcp.port == 443
tcp.dstport == 80
udp.port == 51820
```

### Комбинированные
```
ip.addr == 1.1.1.1 && tcp.port == 443
tcp.port == 443 || tcp.port == 80
!(ip.addr == 192.168.1.1)
ip.addr == 10.0.0.0/24          # подсеть
```

### TCP флаги
```
tcp.flags.syn == 1               # SYN пакеты
tcp.flags.syn == 1 && tcp.flags.ack == 0  # только SYN (новые соединения)
tcp.flags.rst == 1               # RST (сброс соединений)
tcp.flags.fin == 1               # FIN
tcp.analysis.retransmission      # ретрансмиты
tcp.analysis.zero_window         # zero window (перегрузка)
tcp.analysis.duplicate_ack       # дублирующиеся ACK
```

### TLS / HTTPS
```
tls.handshake.type == 1          # ClientHello
tls.handshake.type == 2          # ServerHello
tls.handshake.extensions.server_name == "example.com"  # фильтр по SNI
tls.record.version == 0x0303     # TLS 1.2
tls.record.version == 0x0304     # TLS 1.3
```

### DNS
```
dns.qry.name == "example.com"
dns.qry.type == 1                # A записи
dns.qry.type == 28               # AAAA записи
dns.flags.rcode == 3             # NXDOMAIN (домен не существует)
dns.a == 1.2.3.4                 # ответ содержит IP
```

### Размер пакетов
```
frame.len > 1400                 # большие пакеты (близко к MTU)
frame.len < 100                  # маленькие
tcp.len > 0                      # пакеты с данными (без ACK)
```

---

## Анализ VPN трафика

### Определить WireGuard
```
udp.length == 148                # WireGuard handshake initiation (стандартный)
# или
udp && frame.len == 148
```

AmneziaWG меняет размер handshake — фиксированный паттерн исчезает.

### Определить QUIC / Hysteria2
```
quic
udp.port == 443                  # Hysteria2 обычно на 443/UDP
# Hysteria2 выглядит как обычный QUIC трафик
```

### Анализ энтропии (ищем обфусцированный трафик)
В Wireshark нет встроенного анализа энтропии. Экспортируй в pcap и используй:
```bash
# tshark + скрипт
tshark -r capture.pcap -T fields -e data.data \
  -Y "udp.port == 51820" > raw_data.txt

# Python анализ энтропии
python3 -c "
import math, binascii

def entropy(data):
    if not data: return 0
    freq = {}
    for b in data:
        freq[b] = freq.get(b, 0) + 1
    return -sum(f/len(data) * math.log2(f/len(data)) for f in freq.values())

with open('raw_data.txt') as f:
    for line in f:
        data = bytes.fromhex(line.strip())
        print(f'entropy: {entropy(data):.2f}')
"
# Нормальный трафик: 6-7 бит
# Зашифрованный/сжатый: 7.5-8 бит
```

---

## Захват трафика

### Wireshark — удалённый захват
```bash
# Захват на удалённом хосте, просмотр локально
ssh user@server "tcpdump -i eth0 -nn -w - not port 22" | wireshark -k -i -

# С фильтром
ssh user@server "tcpdump -i eth0 -nn -w - 'port 443'" | wireshark -k -i -
```

### tshark (CLI Wireshark)
```bash
# Захват с фильтром
tshark -i eth0 -f "port 443" -w output.pcap

# Чтение и фильтрация
tshark -r capture.pcap -Y "tls.handshake.type == 1" -T fields \
  -e ip.src -e tls.handshake.extensions.server_name

# Статистика соединений
tshark -r capture.pcap -q -z conv,tcp

# Top talkers
tshark -r capture.pcap -q -z endpoints,ip

# HTTP запросы
tshark -r capture.pcap -Y http.request -T fields \
  -e http.host -e http.request.uri
```

### Ringbuffer — длительный захват без переполнения диска
```bash
tcpdump -i eth0 -w /tmp/capture_%Y%m%d_%H%M%S.pcap \
  -G 300 \         # новый файл каждые 300 секунд
  -W 12 \          # хранить максимум 12 файлов
  -C 100           # максимум 100MB на файл
```

---

## Диагностика TCP проблем

### Ретрансмиты и потери
```
tcp.analysis.retransmission
tcp.analysis.fast_retransmission
tcp.analysis.lost_segment
```
Много ретрансмитов → потери пакетов на пути.

### Задержки
```
tcp.analysis.ack_rtt > 0.1      # RTT больше 100ms
```
В Statistics → TCP Stream Graphs → Round Trip Time — график latency.

### Window size
```
tcp.window_size < 1000          # маленькое окно → тормоз
tcp.analysis.window_full        # окно заполнено
```

### Полный анализ соединения
1. Фильтр: `tcp.stream == N` (N — номер потока)
2. Statistics → TCP Stream Graphs → Time/Sequence (tcptrace)
3. Видно: потери, ретрансмиты, window scaling

---

## Полезные функции Wireshark

### Follow TCP/UDP Stream
ПКМ на пакете → Follow → TCP Stream — видишь весь диалог в читаемом виде.

### Export Objects
File → Export Objects → HTTP — вытащить файлы из захвата.

### IO Graph
Statistics → IO Graph — трафик во времени, можно наложить несколько фильтров.

### Декодирование TLS (если есть ключи)
```bash
# Включить логирование ключей в браузере
export SSLKEYLOGFILE=/tmp/ssl-keys.log
chromium &

# В Wireshark: Edit → Preferences → Protocols → TLS → Pre-Master-Secret log
```

### Colorizing rules
View → Coloring Rules — подсветка пакетов по условию. Полезно для быстрого анализа.

---

## tcpdump — быстрые рецепты

```bash
# Только новые TCP соединения
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn'

# DNS запросы (без ответов)
tcpdump -i eth0 -nn 'udp port 53 and udp[10] & 0x80 = 0'

# Большие пакеты (близко к MTU — ищем фрагментацию)
tcpdump -i eth0 -nn 'len > 1400'

# Трафик кроме SSH
tcpdump -i eth0 -nn 'not port 22'

# Записать и сразу открыть в Wireshark
tcpdump -i eth0 -w - | wireshark -k -i -

# Показать содержимое пакетов в ASCII
tcpdump -i eth0 -A port 80

# Hex + ASCII дамп
tcpdump -i eth0 -XX port 53
```