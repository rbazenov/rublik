# 🚀 Развёртывание «Рублика» на своём сервере — пошаговая инструкция

## 0. Что мы разворачиваем

> В проекте две версии сайта: **`rublik/`** — для российских детей (основная) и `krona/` — ранняя версия для русскоязычных в Швеции. Ниже всё описано для основной; для шведской просто замените путь `rublik/` на `krona/`.

Весь сайт сейчас — **один статический файл** `index.html` (~74 КБ):

- нет сборки (webpack/vite не нужны), нет зависимостей, нет базы данных;
- любой веб-сервер просто отдаёт файл — хватит самого дешёвого VPS;
- резервная копия = копия одного файла;
- обновление сайта = перезалить файл (перезапускать ничего не нужно).

**Что понадобится:**

| Что | Пример | Обязательно? |
|---|---|---|
| Сервер (VPS) с Ubuntu 22.04+ или Debian 12 | Hetzner, Contabo, DigitalOcean, Selectel, Timeweb — хватит 1 vCPU / 1 ГБ RAM | ✅ да |
| SSH-доступ (root или пользователь с sudo) | `ssh root@1.2.3.4` | ✅ да |
| Домен (для HTTPS и красоты адреса) | `rublik.se` | ⬜ рекомендую |

> Все команды ниже выполняются **на сервере** под root. Если входите обычным пользователем — добавляйте `sudo` перед командами.
> Команды, помеченные «на вашем компьютере», выполняются локально, у вас.

---

## Карта вариантов — выберите свой

| Вариант | Кому подходит |
|---|---|
| **A. Nginx** | Классика, основной путь — рекомендую |
| **B. Caddy** | Хочется минимализма: HTTPS получается сам, без certbot |
| **C. Docker** | Сервер уже живёт в Docker |
| **D. Без root** | Быстро показать сайт знакомым (не продакшен) |
| **E. Shared-хостинг / панель** | cPanel, ISPmanager, Plesk — просто загрузить файл |

---

# Вариант A. Nginx на Ubuntu/Debian (рекомендуемый)

### Шаг 1. Подключитесь к серверу

```bash
ssh root@IP_ВАШЕГО_СЕРВЕРА
```

### Шаг 2. Обновите систему и поставьте nginx

```bash
apt update && apt upgrade -y
apt install -y nginx
systemctl enable --now nginx
```

Проверка, что nginx жив:

```bash
curl -I http://localhost
# ожидаем: HTTP/1.1 200 OK
```

### Шаг 3. Создайте каталог сайта

```bash
mkdir -p /var/www/rublik
```

### Шаг 4. Загрузите index.html на сервер

Файл проекта — `rublik/index.html` (скачайте его из воркспейса проекта на свой компьютер).

**На вашем компьютере:**

```bash
scp rublik/index.html root@IP_ВАШЕГО_СЕРВЕРА:/var/www/rublik/
```

Альтернативы:
- **SFTP-клиент** (FileZilla / WinSCP): подключиться к серверу и перетащить файл мышкой в `/var/www/rublik`;
- **Git**: положить проект в репозиторий GitHub/GitLab, затем на сервере `git clone <адрес> /var/www/rublik`.

**На сервере** — права на чтение:

```bash
chmod 755 /var/www/rublik
chmod 644 /var/www/rublik/index.html
```

### Шаг 5. Создайте конфигурацию сайта

```bash
nano /etc/nginx/sites-available/rublik.conf
```

Вставьте (замените `rublik.example.se` на ваш домен; **если домена пока нет** — оставьте `server_name _;` и открывайте сайт по IP):

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name rublik.example.se;

    root /var/www/rublik;
    index index.html;

    # сжатие — страница грузится быстрее
    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;

    # базовые заголовки безопасности
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Сохранить: `Ctrl+O` → `Enter`, выйти: `Ctrl+X`.

### Шаг 6. Включите сайт и перезагрузите nginx

```bash
# убираем дефолтную заглушку «Welcome to nginx»
rm -f /etc/nginx/sites-enabled/default

# включаем наш сайт
ln -s /etc/nginx/sites-available/rublik.conf /etc/nginx/sites-enabled/

# проверка конфигурации и перезагрузка
nginx -t && systemctl reload nginx
```

Если `nginx -t` ругается на синтаксис — проверьте, что скопировали конфиг целиком (особенно точки с запятой).

### Шаг 7. Откройте порты в файрволе

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'    # порты 80 и 443
ufw enable
ufw status
```

⚠️ **Важно:** если сервер в облаке (Hetzner, AWS, Selectel, DigitalOcean…), порты 80/443 нужно открыть ещё и в **панели провайдера** (Security Group / Firewall). Это самая частая причина «у меня не открывается, а curl localhost работает».

### Проверка

Откройте в браузере `http://IP_СЕРВЕРА` (или ваш домен). Должен открыться «Рублик». Пройдите демо-урок до диплома — убедитесь, что игра и квиз работают.

### Шаг 8 (настоятельно рекомендую). HTTPS от Let's Encrypt — бесплатно

1. В панели регистратора домена создайте **A-запись**: `rublik.example.se → IP_СЕРВЕРА` (по желанию ещё `www` CNAME → `rublik.example.se`).
2. Дождитесь обновления DNS (5–30 минут). Проверка на сервере:
   ```bash
   apt install -y dnsutils
   dig +short rublik.example.se    # должен вернуть IP вашего сервера
   ```
3. Получите сертификат:

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d rublik.example.se
```

Certbot сам допишет SSL-настройки в конфиг nginx и настроит редирект на HTTPS. Сертификат бесплатный, на 90 дней, **продлевается автоматически**. Проверка автопродления:

```bash
certbot renew --dry-run
```

### Как обновлять сайт в будущем

Правите `index.html` → перезаливаете. Больше ничего:

```bash
# на вашем компьютере
rsync -av --delete rublik/ root@IP_ВАШЕГО_СЕРВЕРА:/var/www/rublik/
```

Перезагрузка nginx не нужна — статика подхватывается сразу.

---

# Вариант B. Caddy — минимальный путь с автоматическим HTTPS

Файл сайта загружаем так же, как в варианте A (шаги 3–4). Затем:

```bash
# подключаем официальный репозиторий Caddy (Ubuntu/Debian)
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf https://dl.cloudsmith.io/public/caddy/stable/gpg.key | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install -y caddy
systemctl enable --now caddy
```

Откройте `/etc/caddy/Caddyfile`, удалите содержимое и впишите (домен — ваш):

```
rublik.example.se {
    root * /var/www/rublik
    file_server
    encode zstd gzip
}
```

```bash
systemctl reload caddy
```

Готово. HTTPS-сертификат Caddy получит и продлит сам — единственное требование: домен уже указывает на сервер (DNS A-запись, см. Шаг 8 варианта A). Если домена нет, замените первую строку на `:80` — сайт будет доступен по IP.

---

# Вариант C. Docker

Если на сервере уже установлен Docker:

```bash
mkdir -p /opt/rublik/site
# положите index.html в /opt/rublik/site/ (scp, SFTP или git)

docker run -d --name rublik \
  --restart unless-stopped \
  -p 80:80 \
  -v /opt/rublik/site:/usr/share/nginx/html:ro \
  nginx:alpine
```

Сайт — на `http://IP_СЕРВЕРА`. Обновление: заменить файл в `/opt/rublik/site/`, контейнер перезапускать не нужно.

> Для HTTPS в Docker проще всего заменить образ `nginx:alpine` на `caddy` и смонтировать Caddyfile из варианта B, либо держать certbot/Caddy на хосте.

---

# Вариант D. Быстрая проверка без root (не для продакшена!)

Показать сайт знакомым прямо сейчас, ничего не настраивая (на сервере):

```bash
cd rublik
python3 -m http.server 8000
# открыть http://IP_СЕРВЕРА:8000  (порт 8000 должен быть открыт в файрволе)
```

или, если есть Node.js: `npx serve .`

Годно для теста на пару часов. Для реальных пользователей — варианты A/B/C/E.

---

# Вариант E. Shared-хостинг / панель (cPanel, ISPmanager, Plesk)

1. Войдите в панель хостинга → **Файловый менеджер**.
2. Откройте каталог сайта: `public_html`, `www` или `htdocs`.
3. Загрузите `index.html` (если там лежит чужой index — замените).
4. Готово. HTTPS включается тумблером «SSL-сертификат» в панели (Let's Encrypt у большинства хостеров бесплатный).

---

## ✅ Чек-лист после деплоя

- [ ] `curl -I https://ваш-домен` → `HTTP/2 200`
- [ ] Сайт открывается с телефона
- [ ] Демо-урок проходится до конца: игра считает кроны, квиз, диплом печатается
- [ ] Замочек 🔒 в адресной строке (HTTPS работает)
- [ ] Страница весит меньше 100 КБ — грузится мгновенно даже на мобильном интернете

## 🔧 Частые проблемы и решения

| Симптом | Причина и решение |
|---|---|
| «Welcome to nginx» вместо сайта | Не убрали дефолтный сайт или не перезагрузили nginx: `rm -f /etc/nginx/sites-enabled/default && systemctl reload nginx` |
| 403 Forbidden | Нет прав на чтение: `chmod 644 /var/www/rublik/index.html`, каталоги — `755`. Диагностика пути: `namei -l /var/www/rublik/index.html` |
| 404 Not Found | Файл не там, где ждёт nginx: `ls -la /var/www/rublik` и проверьте `root` в конфиге |
| `curl localhost` работает, извне сайт не открывается | Файрвол облака: откройте порты 80/443 в панели провайдера (Security Group / Firewall), затем `ufw status` |
| `nginx: [emerg] bind() to 0.0.0.0:80 failed` | Порт занят другим сервером: `ss -tlnp \| grep :80` — часто это apache → `systemctl disable --now apache2` |
| Certbot: «Failed authorization» | DNS ещё не обновился: `dig +short ваш-домен` — подождите и повторите |
| Правки не видны | Проверьте, что редактируете файл на сервере, а не локально; кэш браузера: `Ctrl+Shift+R` |

## 🔮 Если проект вырастет

Пока «Крона» — статический файл, деплой не меняется вообще. Что изменится в будущем:

- **Регистрация и прогресс учеников** → вариант без серверного кода: Supabase/Firebase (сайт останется статическим, деплой тот же) — или свой бэкенд за nginx;
- **Переход на Next.js / React** → добавится шаг сборки на CI, раздача статики через nginx останется;
- **Онлайн-оплата (Stripe и т.п.)** → секретные ключи живут только на бэкенде, никогда в HTML.

Пришлите, что добавляете, — распишу деплой под новую архитектуру.
