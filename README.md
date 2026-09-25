# TW Images Tool - сервис загрузки и конвертации образов дисков

Веб-сервис для загрузки, хранения и конвертации образов виртуальных дисков
для TimeWeb.Cloud. Позволяет загружать образы с устройства или по ссылке,
конвертировать их между популярными форматами и получать прямые URL для скачивания.

**Демо:** [img.tw-work.ru](https://img.tw-work.ru)

---

## Возможности

- 📤 **Загрузка образов** с устройства (по частям, до 500 ГБ, drag & drop) и по ссылке
- 🔗 **Загрузка по URL** - прямые ссылки, Яндекс.Диск, Google Drive (см. ограничения)
- 💿 **ISO** - загрузка и хранение (без конвертации)
- 🔄 **Конвертация** между форматами: `qcow2` · `vmdk` · `vhd` · `vhdx` · `vdi` · `raw` · `img`
- ⚙️ **Регулируемый лимит** параллельных конвертаций (без перезапуска)
- ⛔ **Отмена** загрузок и конвертаций с остановкой процесса и удалением недописанных файлов
- 🔐 **Парольная защита**: argon2-хеш в БД, подписанная cookie на 31 день, ограничение попыток входа
- 🧹 **Автоочистка** образов старше 3 дней и зависших процессов
- 🌗 **Светлая / тёмная тема**
- 📊 **Мониторинг** активных процессов, прогресса и свободного места в хранилище
- 🌐 **Прямые ссылки** на образы: `https://<домен>/images/имя.формат`

---

## Технологический стек

### Бэкенд
- **Python** + **FastAPI** - API и веб-приложение
- **Uvicorn** - ASGI-сервер (за Nginx)
- **SQLAlchemy** + **PyMySQL** - работа с БД
- **MariaDB / MySQL** - база данных
- **Redis** + **RQ** - две очереди фоновых задач: `conversions` и `downloads`
- **qemu-img** - конвертация образов дисков
- **httpx** - стриминговое скачивание по URL с проверкой адресов (SSRF-защита)
- **argon2-cffi** - хеширование пароля
- **itsdangerous** - подпись cookie-сессий

### Фронтенд
- Ванильный **JavaScript**, **HTML** (Jinja2), **CSS** на кастомных переменных
- SVG-иконки, анимации, адаптивная вёрстка в стиле TimeWeb Cloud

### Инфраструктура
- **Debian 13**
- **systemd** - веб-сервис и два пула воркеров
- **cron** - очистка хранилища и сброс зависших процессов
- **Nginx** - реверс-прокси + прямая раздача образов (с поддержкой Range)
- **Let's Encrypt** (certbot) - HTTPS

---

## Как это работает

```
Браузер → Nginx (443) → Uvicorn (8000) → FastAPI → MySQL + Redis
             │                                        │
             └─ /images/ (раздача файлов)             ├─ imgsvc-conv@N → qemu-img
                                                      └─ imgsvc-dl@N   → httpx
                                  ↓
                          /mnt/storage/images
```

- Недописанные файлы (загрузка или конвертация в процессе) хранятся как `имя.формат.part`.
  В нормальное имя файл переименовывается только после успешного завершения,
  поэтому незаконченные образы не видны в списке и не отдаются по ссылке.
- Каждая конвертация держит слот в Redis с TTL 60 секунд и продлевает его.
  Если воркер упал, слот освобождается сам.
- Если имя уже занято, к нему добавляется номер: `disk(1).qcow2`, `disk(2).qcow2`.

---

## Установка

### 1. Системные зависимости

```bash
apt update && apt install -y \
    python3 python3-venv python3-pip \
    nginx mariadb-server redis-server \
    qemu-utils certbot python3-certbot-nginx git
```

### 2. Проект и окружение

```bash
git clone git@github.com:ВАШ_ЛОГИН/tw-images-tool.git /opt/imgsvc
cd /opt/imgsvc
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 3. Конфигурация

```bash
cp .env.example .env
python -c "import secrets; print(secrets.token_hex(32))"   # значение для SECRET_KEY
```

Заполните `.env` (см. раздел [Конфигурация](#конфигурация-env)).

### 4. БД

```bash
mysql -u root -p -e "CREATE DATABASE imgsvc CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p -e "CREATE USER 'imgsvc'@'localhost' IDENTIFIED BY 'ПАРОЛЬ_БД'; GRANT ALL PRIVILEGES ON imgsvc.* TO 'imgsvc'@'localhost'; FLUSH PRIVILEGES;"
mysql -u imgsvc -p imgsvc < schema.sql
```

Индексы для ускорения запросов по статусу и дате:

```sql
CREATE INDEX idx_uploads_status  ON uploads(status);
CREATE INDEX idx_uploads_created ON uploads(created_at);
CREATE INDEX idx_uploads_name    ON uploads(stored_name);
CREATE INDEX idx_jobs_status     ON jobs(status);
CREATE INDEX idx_jobs_created    ON jobs(created_at);
CREATE INDEX idx_jobs_dst        ON jobs(dst_path(255));
```

> Если индексы уже прописаны в `schema.sql`, этот шаг не нужен.

### 5. Пароль сайта

```bash
python set_password.py 'ВашСложныйПароль'
```

### 6. Хранилище (предварительно подготовленный диск)

```bash
mkdir -p /mnt/storage/{images,uploads,tmp}
chown -R www-data:www-data /mnt/storage
chown -R www-data:www-data /opt/imgsvc
```

> Сетевой диск монтируйте через `/etc/fstab` с опциями `_netdev,nofail`.

### 7. Сервисы systemd

`/etc/systemd/system/imgsvc-web.service`:

```ini
[Unit]
Description=Image Service Web (FastAPI)
After=network.target mariadb.service redis-server.service

[Service]
User=www-data
WorkingDirectory=/opt/imgsvc
Environment=PYTHONPATH=/opt/imgsvc
ExecStart=/opt/imgsvc/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 4
Restart=always

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/imgsvc-conv@.service` - воркеры конвертации:

```ini
[Unit]
Description=Image Service conversion worker %i
After=network.target redis-server.service mariadb.service

[Service]
User=www-data
WorkingDirectory=/opt/imgsvc
Environment=PYTHONPATH=/opt/imgsvc
ExecStart=/opt/imgsvc/venv/bin/python -m app.worker conversions
Restart=always

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/imgsvc-dl@.service` - воркеры загрузок по ссылке:

```ini
[Unit]
Description=Image Service download worker %i
After=network.target redis-server.service mariadb.service

[Service]
User=www-data
WorkingDirectory=/opt/imgsvc
Environment=PYTHONPATH=/opt/imgsvc
ExecStart=/opt/imgsvc/venv/bin/python -m app.worker downloads
Restart=always

[Install]
WantedBy=multi-user.target
```

Запуск:

```bash
systemctl daemon-reload
systemctl enable --now imgsvc-web
for i in $(seq 1 20); do systemctl enable --now imgsvc-conv@$i; done
for i in $(seq 1 4);  do systemctl enable --now imgsvc-dl@$i; done
```

> Число воркеров конвертации должно совпадать с `CONV_WORKERS` в `.env`:
> лимит параллельных конвертаций в интерфейсе нельзя поставить выше этого значения.
> Число воркеров загрузок - это число одновременных загрузок по ссылкам.

Проверка:

```bash
systemctl status imgsvc-web --no-pager
systemctl status 'imgsvc-conv@1' 'imgsvc-dl@1' --no-pager
```

### 8. Nginx + SSL

Получить сертификат:

```bash
certbot certonly --nginx --agree-tos --email admin@email.ru -d domain.ru
```

`/etc/nginx/sites-available/tw-images-tool.conf`:

```nginx
server {
    listen 80;
    server_name domain.ru;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name domain.ru;

    ssl_certificate     /etc/letsencrypt/live/domain.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.ru/privkey.pem;

    client_max_body_size 0;
    client_body_timeout  3600s;
    proxy_read_timeout   3600s;
    proxy_send_timeout   3600s;

    location ~ \.part$ { return 404; }

    location /images/ {
        alias /mnt/storage/images/;
        add_header Accept-Ranges bytes;
    }

    location /static/ {
        alias /opt/imgsvc/static/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_request_buffering off;
    }
}
```

> `X-Real-IP` обязателен: по нему работает ограничение попыток входа
> (5 неудачных попыток за 15 минут с одного IP).

```bash
ln -sf /etc/nginx/sites-available/tw-images-tool.conf /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
systemctl status certbot.timer --no-pager   # автопродление сертификата
```

### 9. Cron

`/opt/imgsvc/cleanup.sh`:

```bash
#!/bin/bash
LOG=/var/log/imgsvc-cleanup.log
cd /opt/imgsvc
echo "=== $(date) ===" >> "$LOG"
/opt/imgsvc/venv/bin/python /opt/imgsvc/cleanup.py >> "$LOG" 2>&1
```

```bash
chmod +x /opt/imgsvc/cleanup.sh
crontab -e
```

```cron
# сброс зависших загрузок и конвертаций - каждые 5 минут
*/5 * * * * cd /opt/imgsvc && /opt/imgsvc/venv/bin/python -m app.maintenance >> /var/log/imgsvc-reaper.log 2>&1

# удаление образов старше 3 дней - ежедневно в 03:00
0 3 * * * /opt/imgsvc/cleanup.sh
```

| Задача | Что делает |
|---|---|
| `app.maintenance` | Переводит в ошибку загрузки и конвертации без активности (после падения или перезапуска воркера) и удаляет их `.part`-файлы |
| `cleanup.py` | Удаляет образы старше 3 дней вместе с записями в БД. Не трогает файлы активных загрузок, а также исходники и результаты идущих конвертаций |

Проверка: `crontab -l`, логи в `/var/log/imgsvc-reaper.log` и `/var/log/imgsvc-cleanup.log`.

---

## Конфигурация (.env)

| Переменная | Описание |
|---|---|
| `DATABASE_URL` | Строка подключения: `mysql+pymysql://imgsvc:ПАРОЛЬ@127.0.0.1/imgsvc` |
| `REDIS_URL` | Адрес Redis: `redis://127.0.0.1:6379/0` |
| `SECRET_KEY` | Секрет для подписи cookie (сгенерировать!) |
| `DATA_DIR` | Каталог хранилища, например `/mnt/storage` |
| `PUBLIC_BASE_URL` | Базовый URL для ссылок на образы, например `https://domain.ru` |
| `CONV_WORKERS` | Число воркеров конвертации и максимум лимита в интерфейсе (по умолчанию `20`) |
| `COOKIE_NAME` | Имя cookie сессии |
| `COOKIE_MAX_AGE` | Срок жизни сессии в секундах (по умолчанию 31 день) |

> ⚠️ Файл `.env` содержит секреты и **не должен** попадать в репозиторий.

---

## Поддерживаемые форматы

| Формат | Ключ qemu-img | Загрузка | Конвертация |
|---|---|:---:|:---:|
| qcow2 | `qcow2` | ✅ | ✅ |
| vmdk | `vmdk` | ✅ | ✅ |
| vhd | `vpc` | ✅ | ✅ |
| vhdx | `vhdx` | ✅ | ✅ |
| vdi | `vdi` | ✅ | ✅ |
| raw / img | `raw` | ✅ | ✅ |
| iso | - | ✅ | ❌ |

> `raw` и `img` - один и тот же формат, конвертация между ними запрещена.
> Для `vhd` и `vhdx` используется `subformat=dynamic`.

---

## Деплой изменений

`/opt/imgsvc/deploy.sh`:

```bash
#!/bin/bash
set -e
cd /opt/imgsvc
git pull origin main
source venv/bin/activate && pip install -r requirements.txt
chown -R www-data:www-data app static
systemctl restart imgsvc-web
for i in $(seq 1 20); do systemctl restart imgsvc-conv@$i; done
for i in $(seq 1 4);  do systemctl restart imgsvc-dl@$i; done
echo "Deploy done: $(date)"
```

```bash
chmod +x /opt/imgsvc/deploy.sh
/opt/imgsvc/deploy.sh
```

> Изменения схемы БД (новые колонки, индексы) `git pull` не применяет - выполняйте их вручную.
> Если перезапустить воркеры во время работы, прерванные процессы станут «Ошибка»
> при следующем запуске `app.maintenance`, а слоты конвертации освободятся примерно через минуту.

---

## Структура проекта

```
/opt/imgsvc/
├── app/
│   ├── main.py          # FastAPI: роуты и API
│   ├── config.py        # конфигурация из .env
│   ├── db.py            # подключение к БД
│   ├── auth.py          # пароль + подписанная cookie
│   ├── uploads.py       # загрузка (по частям + по URL)
│   ├── convert.py       # конвертация через qemu-img, слоты в Redis
│   ├── worker.py        # RQ-воркер (conversions / downloads)
│   ├── maintenance.py   # сброс зависших процессов
│   └── templates/
│       └── index.html
├── static/              # CSS, JS, favicon
├── cleanup.py           # очистка образов старше 3 дней
├── cleanup.sh           # обёртка для cron
├── deploy.sh            # деплой
├── set_password.py      # установка пароля сайта
├── schema.sql           # схема БД
├── requirements.txt
└── .env.example
```

---

## Известные ограничения

- **Яндекс.Диск**: ссылка должна заканчиваться расширением, например
  `https://disk.yandex.ru/d/XXXX.qcow2`. Ссылки без расширения пока отклоняются.
- **Google Drive**: работают только небольшие публичные файлы. Для больших файлов
  Google отдаёт страницу подтверждения, её обход пока не реализован.
- Отмена конвертации в очереди срабатывает сразу, а запущенный `qemu-img`
  останавливается в течение примерно 5 секунд.

---

## Лицензия

© 2026 TimeWeb.Cloud. Все права защищены. Внутренний проект.
Подробнее см. файл [LICENSE](LICENSE).