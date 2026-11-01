# ansible-node-bootstrap

> **EN:** Ansible roles that bring a fresh Ubuntu server to a hardened, ready state and start a proxy node in Docker with one command. Tested on real servers with `--check --diff` and on a clean VPS; passes `ansible-lint` (production profile).

Личный проект по автоматизации развёртывания серверов: шесть Ansible-ролей приводят чистый Ubuntu-сервер в рабочее состояние (пакеты, Docker, защита SSH, файрвол) и запускают в Docker ноду прокси на Xray, управляемую панелью Remnawave. Роли выросли из моей собственной рабочей инфраструктуры.

## Зачем

Ручная настройка серверов приводит к дрейфу конфигурации: серверы незаметно становятся разными. Здесь настройка описана кодом, а роли идемпотентны: повторный запуск ничего не ломает и сообщает `changed=0`, если сервер уже в нужном состоянии.

## Что внутри

| Роль | Что делает |
|---|---|
| `base` | базовые пакеты (curl, ufw, fail2ban, cron, ca-certificates), лимит журнала journald |
| `docker` | официальный репозиторий Docker (ключ проверяется по SHA-256), docker-ce и compose |
| `fail2ban` | защита SSH от перебора, порт SSH для jail |
| `firewall` | ufw: SSH первым правилом, публичные порты, доступ к API ноды только с заданных адресов, входящие запрещены по умолчанию |
| `ssh_hardening` | вход только по ключу, пароли отключены; перед этим кладёт ключ, проверяет, что он на месте, и проверяет конфиг командой `sshd -t` |
| `remnanode` | compose-файл ноды из шаблона, секрет из ansible-vault, запуск контейнера с закреплённой версией образа |

Плейбук `new-node.yml` применяет все шесть ролей по порядку.

## Быстрый старт

1. Установи Ansible и коллекции: `pipx install ansible`, затем `ansible-galaxy collection install -r requirements.yml`.
2. Скопируй примеры и подставь свои значения:
   `cp inventory.example.yml inventory.yml`, `cp group_vars/all.example.yml group_vars/all.yml`, `cp host_vars/node1.example.yml host_vars/node1.yml`.
3. Положи свой публичный SSH-ключ на сервер (первый вход всегда вручную, например `ssh-copy-id`).
4. Зашифруй секрет ноды: `printf '%s' "$KEY" | ansible-vault encrypt_string --stdin-name remnanode_secret_key` и вставь результат в `host_vars/node1.yml`.
5. Сухой прогон: `ansible-playbook new-node.yml -e target=node1 --check`.
6. Настоящий запуск: `ansible-playbook new-node.yml -e target=node1`.

## Безопасность запуска

- Цель обязательна: без `-e target=...` плейбуки ничего не делают (хост `none` не существует). Указывай имя **одного** сервера, а не `all` и не группу.
- Роли меняют SSH и файрвол: ошибка может отрезать доступ. Держи вторую открытую SSH-сессию и сначала проверяй на одноразовом сервере.
- Секреты: в репозиторий попадают только `*.example.yml`, а пароль vault лежит вне проекта.

## Как проверялось

- Сначала роли описали существующую конфигурацию двух рабочих серверов (Ubuntu 24.04 и 26.04), и в режиме `--check` получилось `changed=0`.
- Потом роли по очереди применены на чистом VPS (Ubuntu 24.04), повторный запуск `new-node.yml` дал `changed=0`; нода подключилась к панели.
- `ansible-lint` проходит профиль `production`.

## Чего здесь нет

- Запуска `new-node.yml` одним прогоном на совершенно чистом сервере (роли проверены по отдельности).
- Автоматической проверки в CI и тестов Molecule (планируется).
- Поддержки Debian и архитектур, кроме amd64.
- Автоматического заказа сервера и создания ноды в панели (первый вход и ноду в панели создаю вручную).

## Лицензия

MIT, см. файл `LICENSE`.
