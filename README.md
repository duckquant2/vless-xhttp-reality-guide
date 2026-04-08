# VPN своими руками: VLESS + XHTTP + Reality — от сервера до relay

Пошаговое руководство по развёртыванию собственного VPN на базе Xray-core с маскировкой под обычный HTTPS-трафик. Три уровня: базовый сервер, relay через whitelisted IP и сайт-прикрытие.

---

## Содержание

- [Часть 1 — Боевой VPN-сервер](#часть-1--боевой-vpn-сервер)
- [Часть 2 — Relay через whitelisted IP](#часть-2--relay-через-whitelisted-ip)
- [Часть 3 — Сайт-прикрытие](#часть-3--сайт-прикрытие)
- [Клиенты](#клиенты)

---

## Как это работает

```
Устройство → Боевой VPN-сервер (Европа) → Интернет
```

Если оператор блокирует зарубежные IP:

```
Устройство → Relay (российский whitelisted IP) → Боевой VPN → Интернет
```

**VLESS + Reality** — протокол, который маскирует VPN-трафик под обычное TLS-соединение с легитимным сайтом. DPI-системы видят стандартный HTTPS-хэндшейк — отличить от реального трафика практически невозможно.

**XHTTP** — транспорт, который разбивает VPN-туннель на обычные HTTP-запросы. В сочетании с Reality это выглядит как обычный браузинг.

**Relay** — промежуточный сервер с российским IP из «белого списка» мобильных операторов. Он не расшифровывает трафик, а просто пробрасывает его на боевой сервер за рубежом.

---

## Часть 1 — Боевой VPN-сервер

Это основной сервер за пределами РФ, через который идёт весь трафик.

### Что нужно

VPS у зарубежного хостера (Европа — Германия, Эстония, Латвия, Польша). Минимальный тариф подойдёт — VPN не требует мощных ресурсов. Желательно выбирать небольшого провайдера — меньше риск блокировки всего ASN.

### 1.1 Подключение к серверу

**Mac / Linux:**

```bash
ssh root@ВАШ_IP
```

**Windows** — через [PuTTY](https://putty.org), порт 22.

### 1.2 Установка Xray

```bash
apt update && apt upgrade -y
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
xray version
```

### 1.3 Генерация ключей

```bash
# Пара ключей Reality
xray x25519

# UUID для пользователя
xray uuid

# shortId
openssl rand -hex 8
```

> Сохрани **Private key**, **Public key**, **UUID** и **shortId** — они понадобятся дальше.

Для каждого пользователя генерируй отдельный UUID командой `xray uuid`.

### 1.4 Конфигурация сервера

```bash
nano /usr/local/etc/xray/config.json
```

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "email": "user1",
            "id": "UUID_ПОЛЬЗОВАТЕЛЯ_1"
          },
          {
            "email": "user2",
            "id": "UUID_ПОЛЬЗОВАТЕЛЯ_2"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "xhttpSettings": {
          "path": "/search"
        },
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.delfi.lv:443",
          "xver": 0,
          "serverNames": ["www.delfi.lv"],
          "privateKey": "ВАШ_PRIVATE_KEY",
          "shortIds": ["ВАШ_SHORT_ID"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    }
  ]
}
```

Сохраняем: `Ctrl+X` → `Y` → `Enter`.

> **Про `dest` и `serverNames`:** это домен-прикрытие. Xray имитирует TLS-хэндшейк с этим сайтом. Выбирай крупный зарубежный ресурс с поддержкой TLS 1.3 и X25519 — например `www.delfi.lv`, `www.frankfurt-airport.com`, `www.kernel.org`.
>
> **Важно:** значения `dest` и `serverNames` должны совпадать. Несовпадение — частая причина того, что подключение не работает.

### 1.5 Запуск

```bash
# Проверяем конфиг
xray run -test -c /usr/local/etc/xray/config.json

# Включаем и запускаем
systemctl enable xray
systemctl restart xray
systemctl status xray
```

Должно быть `active (running)`.

### 1.6 Файрвол

```bash
apt install ufw -y
ufw allow ssh
ufw allow 443/tcp
ufw enable
ufw status
```

### 1.7 Ключ подключения для клиента

```
vless://UUID@IP_СЕРВЕРА:443?encryption=none&security=reality&type=xhttp&path=%2Fsearch&mode=auto&sni=www.delfi.lv&fp=chrome&pbk=PUBLIC_KEY&sid=SHORT_ID#MyVPN
```

Параметры:
- `UUID` — UUID пользователя
- `IP_СЕРВЕРА` — IP вашего VPS
- `sni` — домен из `serverNames`
- `pbk` — **Public key** (не Private!)
- `sid` — shortId

### Примечания

- Сертификаты не нужны — Reality использует свой механизм криптографии, имитирующий настоящий TLS.
- Это прямое подключение без relay. Работает на домашнем интернете и у большинства операторов.
- Если у мобильного оператора не работает — скорее всего он блокирует зарубежные IP. В этом случае нужен relay (Часть 2).

---

## Часть 2 — Relay через whitelisted IP

Relay нужен, когда мобильный оператор блокирует прямые подключения к зарубежным IP. Relay принимает трафик на российском IP и пробрасывает его на боевой сервер.

### Что нужно

VPS у российского облачного провайдера, чей ASN входит в «белый список» операторов. Подойдёт самый дешёвый тариф — relay только пробрасывает трафик.

### 2.1 Подключение и установка Xray

Всё то же самое:

```bash
ssh username@IP_RELAY
apt update && apt upgrade -y
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### 2.2 Генерация ключей для relay

```bash
xray uuid        # UUID для каждого клиента
xray x25519      # Своя пара ключей для relay
openssl rand -hex 8  # shortId
```

> У relay свои ключи. Не путай их с ключами боевого сервера.

### 2.3 Конфигурация relay

```bash
nano /usr/local/etc/xray/config.json
```

```json
{
  "log": {
    "loglevel": "info"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {"id": "UUID_КЛИЕНТА_1", "flow": "xtls-rprx-vision", "email": "user1"},
          {"id": "UUID_КЛИЕНТА_2", "flow": "xtls-rprx-vision", "email": "user2"}
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "cloud.cdn.yandex.net:443",
          "xver": 0,
          "serverNames": ["cloud.cdn.yandex.net"],
          "privateKey": "PRIVATE_KEY_RELAY",
          "shortIds": ["SHORT_ID_RELAY"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "IP_БОЕВОГО_СЕРВЕРА",
            "port": 443,
            "users": [
              {
                "id": "UUID_НА_БОЕВОМ_СЕРВЕРЕ",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "xhttp",
        "xhttpSettings": {
          "path": "/search"
        },
        "security": "reality",
        "realitySettings": {
          "fingerprint": "chrome",
          "serverName": "www.delfi.lv",
          "publicKey": "PUBLIC_KEY_БОЕВОГО_СЕРВЕРА",
          "shortId": "SHORT_ID_БОЕВОГО_СЕРВЕРА"
        }
      },
      "tag": "combat-server"
    }
  ]
}
```

> **Inbound (приём от клиентов):** relay использует **VLESS + TCP + Reality** со своими ключами. Для `dest` / `serverNames` подбирай домен, органичный для данного облачного провайдера — например CDN-домен этого же провайдера.
>
> **Outbound (подключение к боевому серверу):** здесь указываются параметры боевого сервера из Части 1 — IP, UUID, publicKey, shortId и path.

### 2.4 Запуск и файрвол

```bash
xray run -test -c /usr/local/etc/xray/config.json
systemctl enable xray
systemctl restart xray
systemctl status xray
```

```bash
apt install ufw -y
ufw allow ssh
ufw allow 443/tcp
ufw enable
```

### 2.5 Ключ подключения через relay

Клиент подключается к **relay**, а не к боевому серверу напрямую:

```
vless://UUID_КЛИЕНТА@IP_RELAY:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=cloud.cdn.yandex.net&fp=chrome&pbk=PUBLIC_KEY_RELAY&sid=SHORT_ID_RELAY&type=tcp#Relay
```

Параметры:
- `UUID_КЛИЕНТА` — UUID из inbound relay
- `IP_RELAY` — IP relay-ноды (российский)
- `sni` — домен из `serverNames` relay
- `pbk` — Public key **relay** (не боевого сервера!)
- `sid` — shortId relay

### Примечания

- Relay не хранит и не расшифровывает трафик — он просто пробрасывает.
- Если боевой сервер упал — relay тоже перестаёт работать. Следи за обоими.
- Для управления: `systemctl stop xray` / `systemctl start xray` / `systemctl disable xray`.

---

## Часть 3 — Сайт-прикрытие

Сайт-прикрытие — это статический сайт, который живёт на relay-ноде. Если провайдер или регулятор начнёт проверять, что висит на IP — он увидит обычный сайт. При необходимости можно быстро переключить relay в «режим сайта», остановив VPN.

### 3.1 Установка Nginx

```bash
apt install nginx -y
mkdir -p /var/www/yoursite
chmod 755 /var/www/yoursite
```

Скопируй файлы статического сайта в `/var/www/yoursite/`.

### 3.2 Конфигурация Nginx

```bash
nano /etc/nginx/sites-available/yoursite
```

```nginx
# === Режим VPN (основной) ===

server {
    listen 80;
    server_name yourdomain.com;
    root /var/www/yoursite;
    index index.html;

    location /.well-known/acme-challenge/ {
        root /var/www/yoursite;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 8080;
    server_name _;
    root /var/www/yoursite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

# === Режим Сайта (при необходимости — раскомментировать, закомментировать блок выше) ===

#server {
#    listen 80;
#    server_name yourdomain.com;
#    return 301 https://$host$request_uri;
#}

#server {
#    listen 443 ssl http2;
#    server_name yourdomain.com;
#    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
#    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
#    root /var/www/yoursite;
#    index index.html;
#    location / {
#        try_files $uri $uri/ =404;
#    }
#}
```

```bash
ln -s /etc/nginx/sites-available/yoursite /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

### 3.3 DNS и SSL

Добавь A-запись для домена, указывающую на IP relay-сервера.

```bash
# Проверяем DNS
dig yourdomain.com +short

# Открываем порт 80 для certbot
ufw allow 80/tcp
ufw reload

# Получаем сертификат
apt install certbot python3-certbot-nginx -y
certbot --nginx -d yourdomain.com

# Проверяем автопродление
certbot renew --dry-run
```

### 3.4 Связка с Xray

В конфиге Xray (`/usr/local/etc/xray/config.json`) в секции `realitySettings` добавь fallback на Nginx:

```json
"realitySettings": {
    ...
    "shortIds": ["..."],
    "fallback": {
        "dest": 8080
    }
}
```

Теперь при обращении к серверу без валидного VPN-ключа — Xray отдаёт сайт через Nginx на порту 8080.

```bash
xray run -test -c /usr/local/etc/xray/config.json
systemctl restart xray
```

### 3.5 Переключение режимов

#### VPN → Сайт

```bash
# Останавливаем VPN
systemctl stop xray

# В конфиге Nginx: комментируем «Режим VPN», раскомментируем «Режим Сайта»
nano /etc/nginx/sites-available/yoursite

# Перезапускаем
nginx -t
systemctl reload nginx
```

#### Сайт → VPN

```bash
# В конфиге Nginx: комментируем «Режим Сайта», раскомментируем «Режим VPN»
nano /etc/nginx/sites-available/yoursite

# Перезапускаем
nginx -t
systemctl reload nginx
systemctl start xray

# Проверяем
systemctl status nginx
systemctl status xray
```

---

## Клиенты

| Платформа | Приложение | Ссылка |
|-----------|-----------|--------|
| iPhone | v2RayTun | [App Store](https://apps.apple.com/app/v2raytun/id6476628951) |
| Android | v2rayNG | [Google Play](https://play.google.com/store/apps/details?id=com.v2ray.ang) |
| Windows | Hiddify | [GitHub](https://github.com/hiddify/hiddify-next/releases) |
| Mac | Hiddify | [GitHub](https://github.com/hiddify/hiddify-next/releases) |

Добавляй ключ подключения через QR-код или вставкой строки.

---

## FAQ

**Нужны ли SSL-сертификаты для VPN?**
Нет. Reality использует собственную криптографию, имитирующую TLS. Сертификаты нужны только для сайта-прикрытия.

**Можно ли использовать боевой сервер без relay?**
Да. Relay нужен только если мобильный оператор блокирует зарубежные IP. На домашнем интернете обычно работает напрямую.

**Сколько пользователей можно подключить?**
Ограничений в протоколе нет. Для каждого пользователя генерируй отдельный UUID (`xray uuid`) и добавляй в массив `clients`.

**Relay видит мой трафик?**
Нет. Relay пробрасывает зашифрованный поток, не расшифровывая его.

**Что если боевой сервер заблокируют?**
Арендуешь новый VPS, разворачиваешь по Части 1 и обновляешь outbound на relay. Клиенты продолжают подключаться к тому же relay — менять у них ничего не нужно (если relay-ключи не менялись).

---
