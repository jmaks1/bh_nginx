# Nginx config (prod)

Структура для деплоя на сервер (Debian: `/etc/nginx/`). Рефакторинг в 3 фазы: snippets → upstream’ы → шаблон для новых доменов. Поведение 1-в-1 как до изменений.

## Структура

```
conf.d/
  upstream.conf       # frontend_biletoffhub, api_biletoffhub
snippets/
  proxy-common.conf   # общие proxy headers
  proxy-ws.conf       # WebSocket headers
sites-available/
  biletoffhub.conf
  api.biletoffhub.conf
  vsebilety.pro.conf
  api.vsebilety.pro.conf
templates/
  https-proxy.conf    # шаблон для новых доменов ({{DOMAIN}}, {{CERT_DOMAIN}}, {{UPSTREAM}})
  domain.conf         # старый шаблон (плейсхолдеры)
```

## Сертификаты (SSL)

Сертификаты **не хранятся в репозитории** (в `.gitignore` — `*.pem`, `*.key`, `/etc/letsencrypt/`). Они лежат только на сервере и управляются Certbot.

- **Путь на сервере:** `/etc/letsencrypt/live/<домен>/` — там `fullchain.pem` и `privkey.pem`. В конфигах сайтов указаны именно эти пути.
- **Новый домен:** сначала получить сертификат, потом деплоить конфиг. Пример:
  ```bash
  certbot certonly --nginx -d example.com -d www.example.com -d api.example.com
  ```
  Certbot сам настроит временный HTTP и выдаст сертификат. После этого можно включить наш конфиг из `sites-available/`.
- **Продление:** `certbot renew` (часто уже висит в cron). Конфиги не трогаем.

## Симлинки (sites-available → sites-enabled)

На Debian/Ubuntu nginx подключает только то, что лежит в **sites-enabled/** (в `nginx.conf`: `include /etc/nginx/sites-enabled/*`). Папка **sites-available/** — просто хранилище всех конфигов: включённых и выключенных.

**Симлинк** — это «ссылка» на файл: в `sites-enabled/` лежит не копия конфига, а указатель на файл в `sites-available/`. Nginx читает конфиг по этой ссылке и подхватывает его.

Зачем так делать:
- **Включить сайт** — создать симлинк: конфиг сразу участвует в конфигурации.
- **Выключить сайт** — удалить симлинк (файл в `sites-available/` остаётся, его можно снова включить).
- Один конфиг в одном месте — правки только в `sites-available/`, перезагрузка nginx подхватывает изменения.

**Включить сайт:**
```bash
ln -s /etc/nginx/sites-available/имя.conf /etc/nginx/sites-enabled/имя.conf
```

**Посмотреть, какие сайты включены** (в `sites-enabled/` обычно только симлинки):
```bash
ls -la /etc/nginx/sites-enabled/
```
В выводе видно имя файла и куда он ссылается (например `biletoffhub.conf -> /etc/nginx/sites-available/biletoffhub.conf`).

**Выключить сайт** — удалить симлинк (конфиг в `sites-available/` не трогается):
```bash
rm /etc/nginx/sites-enabled/имя.conf
```
После изменений: `nginx -t && systemctl reload nginx`.

Прописывать симлинки вручную нужно потому, что nginx не смотрит в `sites-available/` — он смотрит только в `sites-enabled/`. Без симлинка конфиг в `sites-available/` просто не будет загружен.

## Синхронизация репозитория с сервером

Репо — источник правды; на сервер нужно только подвозить файлы. Два рабочих варианта.

**Вариант 1: rsync с локальной машины** (проще всего)

С твоего компьютера (из корня репо, где есть папка `infra/nginx/`):
```bash
rsync -avz --delete infra/nginx/conf.d/   user@SERVER:/etc/nginx/conf.d/
rsync -avz --delete infra/nginx/snippets/ user@SERVER:/etc/nginx/snippets/
rsync -avz --delete infra/nginx/sites-available/ user@SERVER:/etc/nginx/sites-available/
```
- `user@SERVER` — заменить на своего пользователя и хост (или использовать SSH-алиас из `~/.ssh/config`).
- `--delete` — на сервере в этих папках удалятся файлы, которых нет в репо (осторожно: в `sites-enabled/` не трогаем, только conf.d, snippets, sites-available).

Симлинки в `sites-enabled/` создаются один раз вручную и при rsync не перезатираются (мы не синхронизируем `sites-enabled/`).

**Вариант 2: git на сервере**

На сервере клонировать репо в отдельную директорию (не в `/etc/nginx/`), например:
```bash
git clone <url-репо> /opt/bh_nginx
```
При обновлении конфигов:
```bash
cd /opt/bh_nginx && git pull
sudo cp -r infra/nginx/conf.d/*    /etc/nginx/conf.d/
sudo cp -r infra/nginx/snippets/* /etc/nginx/snippets/
sudo cp -r infra/nginx/sites-available/* /etc/nginx/sites-available/
```
Симлинки в `sites-enabled/` создаются один раз и не трогаются при копировании.

Плюс rsync — не нужен git на сервере и не нужен доступ к репо с сервера. Плюс git на сервере — можно делать `git pull` по SSH и один раз настроить скрипт деплоя. Для одного сервера чаще достаточно rsync.

## Деплой

1. Скопировать `conf.d/`, `snippets/`, `sites-available/` в `/etc/nginx/` (вручную или через синхронизацию выше).
2. Включить сайты симлинками: `ln -s /etc/nginx/sites-available/ИМЯ.conf /etc/nginx/sites-enabled/`.
3. На сервере: заменить/объединить старый `conf.d/upstream.conf` с этим.
4. `nginx -t` → затем `systemctl reload nginx` (не restart).

## Новые домены

Использовать шаблон `templates/https-proxy.conf`: подставить `{{DOMAIN}}`, `{{CERT_DOMAIN}}`, `{{UPSTREAM}}`. Старые конфиги не трогать.

## Важно

- На сервере `snippets/` — по путям `snippets/...` (от корня nginx, т.е. `/etc/nginx/snippets/`).
- Certbot и SSL не меняем; if’ы от Certbot и default не трогаем в этой фазе.
