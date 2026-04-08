# VPN своими руками: VLESS + XHTTP + Reality для самых маленьких

Пошаговое руководство по развёртыванию собственного VPN на базе Xray-core с маскировкой под обычный HTTPS-трафик. Четыре уровня: базовый сервер, туннель через белый IP, домен и сайт-прикрытие.

---

## Содержание

- [Часть 1 - Боевой VPN-сервер](#part-1)
- [Часть 2 - Туннель через белый IP](#part-2)
- [Часть 3 - Домен](#part-3)
- [Часть 4 - Сайт-прикрытие](#part-4)
- [Клиенты](#clients)
- [FAQ](#faq)

---

## Как это работает

```
Устройство → Боевой VPN-сервер (Европа) → Интернет
```

Если оператор блокирует зарубежные IP:

```
Устройство → Туннель (российский белый IP) → Боевой VPN → Интернет
```

**VLESS + Reality** - протокол, который маскирует VPN-трафик под обычное TLS-соединение с легитимным сайтом. DPI-системы видят стандартный HTTPS-хэндшейк.

**XHTTP** - транспорт, который разбивает VPN-туннель на обычные HTTP-запросы. В сочетании с Reality это выглядит как обычный браузинг.

**Туннель** - промежуточный сервер с российским IP из белого списка мобильных операторов. Он не расшифровывает трафик, а просто пробрасывает его на боевой сервер за рубежом.

---

<a id="part-1"></a>
## Часть 1 - Боевой VPN-сервер

Это основной сервер за пределами РФ, через который идёт весь трафик.

### Что нужно

VPS у зарубежного хостера (Европа - Германия, Эстония, Латвия, Польша). Минимальный тариф подойдёт - VPN не требует мощных ресурсов. Желательно выбирать небольшого провайдера - меньше риск блокировки всего ASN.

### 1.1 Подключение к серверу

**Mac / Linux:**

```bash
ssh root@ВАШ_IP
```

**Windows** - через [PuTTY](https://putty.org), порт 22.

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

> Сохрани **Private key**, **Public key**, **UUID** и **shortId** - они понадобятся дальше.

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
          "dest": "www.wp.pl:443",
          "xver": 0,
          "serverNames": ["www.wp.pl"],
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

> **Про `dest` и `serverNames`:** это домен-прикрытие. Xray имитирует TLS-хэндшейк с этим сайтом. Выбирай крупный зарубежный ресурс с поддержкой TLS 1.3 и X25519 - например `www.wp.pl`, `www.kernel.org`.
>
> **Важно:** значения `dest` и `serverNames` должны совпадать. Несовпадение - частая причина того, что подключение не работает.

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
vless://UUID@IP_СЕРВЕРА:443?encryption=none&security=reality&type=xhttp&path=%2Fsearch&mode=auto&sni=www.wp.pl&fp=chrome&pbk=PUBLIC_KEY&sid=SHORT_ID#MyVPN
```

Параметры:
- `UUID` - UUID пользователя
- `IP_СЕРВЕРА` - IP вашего VPS (или домен - см. Часть 3)
- `sni` - домен из `serverNames`
- `pbk` - **Public key** (не Private!)
- `sid` - shortId

### Примечания

- Сертификаты не нужны - Reality использует свой механизм криптографии, имитирующий настоящий TLS.
- Это прямое подключение без туннеля. Работает на домашнем интернете и у большинства операторов.
- Если у мобильного оператора не работает - скорее всего он блокирует зарубежные IP. В этом случае нужен туннель (Часть 2).

---

<a id="part-2"></a>
## Часть 2 - Туннель через белый IP

Туннель нужен, когда мобильный оператор вводит режим белых списков. Туннель принимает трафик на российском IP и пробрасывает его на боевой сервер.

### Что нужно

VPS у российских провайдеров, чей ASN входит в белый список операторов (они точно есть у компаний на букву Я и на букву V). Подойдёт самый дешёвый сервер - туннель только пробрасывает трафик.

### 2.1 Подключение и установка Xray

Всё то же самое:

```bash
ssh username@IP_ТУННЕЛЯ
apt update && apt upgrade -y
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### 2.2 Генерация ключей

```bash
xray uuid        # UUID для каждого клиента
xray x25519      # Своя пара ключей для туннеля
openssl rand -hex 8  # shortId
```

> У туннеля свои ключи. Не путай их с ключами боевого сервера.

### 2.3 Конфигурация туннеля

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
          "dest": "max.ru:443",
          "xver": 0,
          "serverNames": ["max.ru"],
          "privateKey": "PRIVATE_KEY_ТУННЕЛЯ",
          "shortIds": ["SHORT_ID_ТУННЕЛЯ"]
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
            "address": "vpn.example.com",
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
          "serverName": "www.wp.pl",
          "publicKey": "PUBLIC_KEY_БОЕВОГО_СЕРВЕРА",
          "shortId": "SHORT_ID_БОЕВОГО_СЕРВЕРА"
        }
      },
      "tag": "combat-server"
    }
  ]
}
```

> **Inbound (приём от клиентов):** туннель использует **VLESS + TCP + Reality** со своими ключами. Для `dest` / `serverNames` подбирай домен, органичный для данного облачного провайдера - например CDN-домен этого же провайдера.
>
> **Outbound (подключение к боевому серверу):** здесь указываются параметры боевого сервера из Части 1 - домен (или IP), UUID, publicKey, shortId и path. Если используешь домен из Части 3, при смене боевого сервера туннель не нужно перенастраивать.

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

### 2.5 Ключ подключения через туннель

Клиент подключается к **туннелю**, а не к боевому серверу напрямую:

```
vless://UUID_КЛИЕНТА@IP_ТУННЕЛЯ:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=max.ru&fp=chrome&pbk=PUBLIC_KEY_ТУННЕЛЯ&sid=SHORT_ID_ТУННЕЛЯ&type=tcp#Relay
```

Параметры:
- `UUID_КЛИЕНТА` - UUID из inbound туннеля
- `IP_ТУННЕЛЯ` - IP туннеля (или домен - см. Часть 3)
- `sni` - домен из `serverNames` туннеля
- `pbk` - Public key **туннеля** (не боевого сервера!)
- `sid` - shortId туннеля

### Примечания

- Туннель не хранит и не расшифровывает трафик - он просто пробрасывает.
- Если боевой сервер упал - туннель тоже перестаёт работать. Следи за обоими.
- Для управления: `systemctl stop xray` / `systemctl start xray` / `systemctl disable xray`.

---

<a id="part-3"></a>
## Часть 3 - Домен

Если в ключе подключения указан IP-адрес, то при блокировке этого IP придётся менять ключи у всех клиентов. Домен решает эту проблему: ты меняешь A-запись в DNS на новый IP - и все клиенты продолжают работать без перенастройки.

Это работает и для боевого сервера, и для туннеля.

### 3.1 Покупка домена

Подойдёт любой дешёвый домен в зонах `.com`, `.net`, `.top`, `.xyz` и т.д. Главное - чтобы у регистратора была возможность управлять DNS-записями.

> Не используй `.ru` / `.рф` - эти зоны управляются изнутри РФ.

### 3.2 Настройка DNS

Создай A-запись, указывающую на IP твоего сервера:

```
Тип: A
Имя: vpn (или @ для корневого домена)
Значение: IP_ВАШЕГО_СЕРВЕРА
TTL: 300 (5 минут - для быстрого переключения)
```

Пример: если домен `example.com` и запись `vpn`, то адрес будет `vpn.example.com`.

Проверяем:

```bash
dig vpn.example.com +short
# Должен вернуть IP вашего сервера
```

> **TTL 300** - короткий TTL нужен, чтобы при смене IP новый адрес подхватился быстро. По умолчанию у многих регистраторов TTL стоит 3600 (1 час) или выше - поменяй на 300.

### 3.3 Ключ подключения через домен

Теперь в ключе вместо IP указываешь домен:

**Для боевого сервера:**

```
vless://UUID@vpn.example.com:443?encryption=none&security=reality&type=xhttp&path=%2Fsearch&mode=auto&sni=www.wp.pl&fp=chrome&pbk=PUBLIC_KEY&sid=SHORT_ID#MyVPN
```

**Для туннеля:**

```
vless://UUID_КЛИЕНТА@relay.example.com:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=max.ru&fp=chrome&pbk=PUBLIC_KEY_ТУННЕЛЯ&sid=SHORT_ID_ТУННЕЛЯ&type=tcp#Relay
```

> Можно использовать один домен с разными поддоменами: `vpn.example.com` для боевого сервера, `relay.example.com` для туннеля.

### 3.4 Что делать при блокировке IP

1. Арендуешь новый VPS
2. Разворачиваешь Xray по Части 1 (с теми же ключами, UUID и shortId)
3. Меняешь A-запись в DNS на новый IP
4. Ждёшь 5-10 минут (TTL)
5. Клиенты начинают работать без перенастройки

> **Важно:** при переезде на новый сервер используй те же **Private key**, **shortIds** и **UUID** клиентов - тогда ключи подключения останутся полностью валидными. Меняется только IP за доменом.

### 3.5 Домен для туннеля в outbound

Если у боевого сервера есть домен, его же можно использовать в outbound туннеля - тогда при переезде боевого сервера в конфиге туннеля не нужно будет ничего менять:

```json
"outbounds": [
  {
    "protocol": "vless",
    "settings": {
      "vnext": [
        {
          "address": "vpn.example.com",
          "port": 443,
          ...
        }
      ]
    },
    ...
  }
]
```

---

<a id="part-4"></a>
## Часть 4 - Сайт-прикрытие

Сайт-прикрытие - это статический сайт, который живёт на VPS туннеля. Если провайдер или регулятор начнёт проверять, что висит на IP - он увидит обычный сайт. При необходимости можно быстро переключиться в режим сайта, остановив VPN.

### 4.1 Установка Nginx

```bash
apt install nginx -y
mkdir -p /var/www/yoursite
chmod 755 /var/www/yoursite
```

Скопируй файлы статического сайта в `/var/www/yoursite/`.

### 4.2 Конфигурация Nginx

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

# === Режим Сайта (при необходимости - раскомментировать, закомментировать блок выше) ===

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

### 4.3 DNS и SSL

Добавь A-запись для домена, указывающую на IP туннеля.

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

### 4.4 Связка с Xray

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

Теперь при обращении к серверу без валидного VPN-ключа - Xray отдаёт сайт через Nginx на порту 8080.

```bash
xray run -test -c /usr/local/etc/xray/config.json
systemctl restart xray
```

### 4.5 Переключение режимов

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

<a id="clients"></a>
## Клиенты

| Платформа | Приложение | Ссылка |
|-----------|-----------|--------|
| iPhone | v2RayTun | [App Store](https://apps.apple.com/app/v2raytun/id6476628951) |
| Android | v2rayNG | [Google Play](https://play.google.com/store/apps/details?id=com.v2ray.ang) |
| Windows | Throne | [GitHub](https://github.com/throneproj/Throne/releases) |
| Mac | Throne | [GitHub](https://github.com/throneproj/Throne/releases) |

Добавляй ключ подключения через QR-код или вставкой строки.

---

<a id="faq"></a>
## FAQ

**Нужны ли SSL-сертификаты для VPN?**
Нет. Reality использует собственную криптографию, имитирующую TLS. Сертификаты нужны только для сайта-прикрытия.

**Можно ли использовать боевой сервер без туннеля?**
Да. Туннель нужен только если мобильный оператор вводит режим белых списков. На домашнем интернете обычно работает напрямую.

**Сколько пользователей можно подключить?**
Ограничений в протоколе нет. Для каждого пользователя генерируй отдельный UUID (`xray uuid`) и добавляй в массив `clients`.

**Туннель видит мой трафик?**
Нет. Туннель пробрасывает зашифрованный поток, не расшифровывая его.

**Что если боевой сервер заблокируют?**
Если используешь домен (Часть 3) - арендуешь новый VPS, разворачиваешь Xray с теми же ключами, меняешь A-запись в DNS. Клиенты продолжают работать без перенастройки. Если без домена - придётся обновить ключи у всех клиентов вручную.

**Зачем домен, если есть туннель?**
Домен и туннель решают разные задачи. Домен позволяет безболезненно менять сервер при блокировке IP. Туннель нужен для работы с мобильным интернетом в режиме белых списков. Лучше использовать оба.

---
