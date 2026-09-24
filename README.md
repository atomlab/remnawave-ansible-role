# remnawave-ansible-role

Ansible-роль для развёртывания стека [Remnawave](https://remna.st): панель, страница подписок, нода Xray — плюс опциональный веб-редактор конфигов [Xray Config UI Editor](https://github.com/bropines/xray-config-ui-editor) на домене панели.

Всё крутится в docker (docker compose) под systemd-юнитами, TLS terminates angie с автоматическим Let's Encrypt через встроенный ACME-клиент angie.

## Структура

Роль состоит из трёх независимых входных точек — используйте только нужные:

| Входная точка | Что ставит | Хост |
|---|---|---|
| `remnawave/panel` | Панель (backend + Postgres 17 + Valkey), reverse-proxy angie с ACME-сертификатами, опционально Xray Config UI на `https://<домен>/xray-ui/` | сервер панели |
| `remnawave/sub` | Страница подписок (`remnawave/subscription-page`) | сервер панели или отдельный |
| `remnawave/node` | Нода `remnawave/node` | каждый сервер нод |

## Требования

- Ubuntu/Debian с установленными **docker** и **docker compose plugin**;
- домены, направленные A/AAAA-записями на сервер панели (сертификаты выпускаются автоматически).

## Быстрый старт (панель + xray-ui)

1. Склонируйте репозиторий рядом со своим playbook:

```bash
git clone <этот репозиторий> roles/remnawave-ansible-role
```

2. Заполните переменные (см. `playbook.yaml` и таблицы ниже): `remna_domain`, `remna_sub_domain` и т.д.

3. Запустите:

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

4. Роль создаст `{{ remna_workdir }}` (`/opt/remnawave/panel`), положит туда `docker-compose.yml`, конфиги и **`env.sample`** — рабочий `.env` роль намеренно не генерирует, его заполняют руками один раз:

```bash
cd /opt/remnawave/panel
cp env.sample .env

# секреты
sed -i "s/^JWT_AUTH_SECRET=.*/JWT_AUTH_SECRET=$(openssl rand -hex 64)/" .env
sed -i "s/^JWT_API_TOKENS_SECRET=.*/JWT_API_TOKENS_SECRET=$(openssl rand -hex 64)/" .env
sed -i "s/^METRICS_PASS=.*/METRICS_PASS=$(openssl rand -hex 24)/" .env
sed -i "s/^WEBHOOK_SECRET_HEADER=.*/WEBHOOK_SECRET_HEADER=$(openssl rand -hex 32)/" .env

# пароль Postgres: должен совпадать с DATABASE_URL
pw=$(openssl rand -hex 24)
sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$pw/" .env
sed -i "s|^\(DATABASE_URL=\"postgresql://postgres:\)[^@\"]*\(@.*\)|\1$pw\2|" .env
```

Затем: `systemctl start remna-panel`. Подробности про переменные окружения — в [документации Remnawave](https://remna.st/docs/install/environment-variables) и в `roles/remnawave/panel/README.md`.

## Пример playbook

См. [playbook.yaml](playbook.yaml) — панель с xray-ui, страница подписок и одна нода.

## Переменные

### `remnawave/panel` (defaults/main.yml)

| Переменная | По умолчанию | Описание |
|---|---|---|
| `remna_workdir` | `/opt/remnawave/panel` | Рабочий каталог: compose, конфиги proxy, данные БД |
| `remna_release` | `3.4.4` | Тег образа `remnawave/backend` |
| `remna_domain` | `panel.example.com` | Домен панели (по нему angie выпускает сертификат) |
| `remna_sub_same_host_enabled` | `false` | Крутить ли страницу подписок на том же сервере (добавляет server-блок в angie) |
| `remna_sub_domain` | `sub.example.com/api/sub` | Домен подписок |
| `remna_sub_upstream` | `127.0.0.1:3010` | Адрес subscription-page |
| `reman_custom_sub_prefix` | `""` | CUSTOM_SUB_PREFIX |
| `remna_xray_ui_release` | `main` | Тег/ветка [xray-config-ui-editor](https://github.com/bropines/xray-config-ui-editor), из которой собирается образ |

### `remnawave/sub` (defaults/main.yml)

| Переменная | По умолчанию |
|---|---|
| `remnasub_workdir` | `/opt/remnawave/sub` |
| `remnasub_release` | `8.0.0` |
| `remnasub_panel_url` | `https://panel.example.com` |
| `remnasub_port` | `3010` |
| `remnasub_custom_sub_prefix` | `""` |
| `remnasub_meta_title` / `remnasub_meta_desc` | заголовок/описание страницы |
| `remnasub_remnawave_api_token` | `secret` — API-токен панели |

### `remnawave/node` (defaults/main.yml)

| Переменная | По умолчанию |
|---|---|
| `remnanode_workdir` | `/opt/remnawave/node` |
| `remnanode_release` | `2.8.0` |
| `remnanode_node_port` | `2222` |

Ноде, как и панели, нужен файл `.env` (переменные ноды — см. [документацию](https://remna.st)): роль создаёт каталог и compose, `.env` заполняется вручную в `{{ remnanode_workdir }}/.env`. Выпуск TLS-сертификатов для ноды — вне скоупа роли: подключите свои сертификаты через `volumes` в `roles/remnawave/node/templates/docker-compose.yml`.

## Xray Config UI Editor (`/xray-ui/`)

Роль собирает из исходников docker-образ со статикой редактора и отдаёт его на `https://<remna_domain>/xray-ui/`. Так как UI живёт на домене панели, синк с Remnawave работает без CORS-настроек.

- релиз UI задаётся `remna_xray_ui_release` (тег или ветка репозитория редактора);
- при смене релиза пересоберите контейнер: `docker compose build xray-ui && docker compose up -d xray-ui`;
- конфиг веб-сервера контейнера — обычный файл на хосте `{{ remna_workdir }}/xray-ui/angie.conf`, монтируется в контейнер read-only; правится на месте и применяется `docker compose restart xray-ui`.

## Обновления

Меняются соответствующие `*_release` переменные, затем `ansible-playbook ... --check --diff` и прогон. Юниты (`remna-panel`, `remna-sub`, `remna-node`) делают `docker compose down && up`, поэтому новые образы подхватываются при рестарте сервиса.

## Лицензия

MIT — см. [LICENSE](LICENSE).
