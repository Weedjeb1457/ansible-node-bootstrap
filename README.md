# ansible-node-bootstrap
![lint](https://github.com/Weedjeb1457/ansible-node-bootstrap/actions/workflows/lint.yml/badge.svg)

> **EN:** Ansible roles that bring a fresh Ubuntu server to a hardened state and run a proxy node in Docker with a single playbook. Passes `ansible-lint` (production profile).

Личный проект по автоматизации настройки серверов: шесть Ansible-ролей приводят чистый Ubuntu-сервер в рабочее состояние (пакеты, Docker, защита SSH, файрвол) и запускают в Docker ноду прокси на Xray под управлением панели Remnawave. Роли составлены по конфигурации моих рабочих серверов.

## Роли

| Роль | Назначение |
|---|---|
| `base` | базовые пакеты (curl, ufw, fail2ban, cron, ca-certificates), лимит журнала journald |
| `docker` | официальный репозиторий Docker (ключ проверяется по SHA-256), docker-ce и compose |
| `fail2ban` | защита SSH от перебора, порт SSH для jail |
| `firewall` | ufw: SSH первым правилом, публичные порты, доступ к API ноды только с заданных адресов, входящие запрещены по умолчанию |
| `ssh_hardening` | вход только по ключу; перед отключением паролей ключ добавляется и проверяется, конфиг проверяется командой `sshd -t` |
| `remnanode` | compose-файл ноды из шаблона, секрет из ansible-vault, запуск контейнера с закреплённой версией образа |

Плейбук `new-node.yml` применяет все шесть ролей по порядку. Роли идемпотентны: повторный запуск сообщает `changed=0`, если сервер уже в нужном состоянии.

## Использование

Установка Ansible и коллекций:

```
pipx install ansible
ansible-galaxy collection install -r requirements.yml
```

Подготовка переменных из примеров:

```
cp inventory.example.yml inventory.yml
cp group_vars/all.example.yml group_vars/all.yml
cp host_vars/node1.example.yml host_vars/node1.yml
```

Секрет ноды хранится в ansible-vault. Значение шифруется командой

```
printf '%s' "$KEY" | ansible-vault encrypt_string --stdin-name remnanode_secret_key
```

и вставляется в `host_vars/node1.yml`. Первый вход на сервер выполняется вручную (например, `ssh-copy-id`), после этого публичный ключ закрепляется ролью `ssh_hardening`.

Сухой прогон и запуск:

```
ansible-playbook new-node.yml -e target=node1 --check
ansible-playbook new-node.yml -e target=node1
```

## Замечания по безопасности

- Цель запуска задаётся переменной `target`. Без неё плейбуки ничего не выполняют (хоста `none` не существует). В качестве цели указывается имя одного сервера, а не `all` и не группа.
- Роли меняют SSH и файрвол, поэтому возможна потеря доступа. Перед запуском на рабочем сервере используются сухой прогон и вторая открытая SSH-сессия.
- В репозитории хранятся только файлы `*.example.yml`. Пароль vault лежит вне проекта.

## Как проверялось

- Роли описывают конфигурацию двух рабочих серверов (Ubuntu 24.04 и 26.04): в режиме `--check` результат `changed=0`.
- Роли по очереди применены на чистом VPS (Ubuntu 24.04), после этого `new-node.yml` на том же сервере дал `changed=0`; нода подключилась к панели. Полный прогон `new-node.yml` одним запуском на совершенно чистом сервере не выполнялся.
- `ansible-lint` проходит профиль `production`, проверка запускается в GitHub Actions при каждом коммите.
- После переименования переменных `new-node.yml` повторно проверен в режиме `--check` на двух рабочих серверах: `changed=0`.

## Лицензия

MIT, см. файл `LICENSE`.
