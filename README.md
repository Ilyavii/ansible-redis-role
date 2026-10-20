# Ansible-роль для установки и настройки Redis

Идемпотентная роль устанавливает Redis (только Debian/Ubuntu), создаёт каталог данных,
генерирует `redis.conf` через Jinja2-шаблон и управляет systemd-сервисом.
Redis перезапускается только при изменении конфигурации (через handler).
Пароль Redis хранится в Ansible Vault, поддерживаются окружения `dev` и `prod`.

## Структура проекта

```
.
├── playbook.yaml                  # Главный плейбук (единый для всех окружений)
├── README.md
├── inventories/
│   ├── dev/
│   │   ├── hosts.ini              # Хосты dev-окружения
│   │   └── group_vars/
│   │       ├── redis.yml          # Переменные dev (порт, dir, ...)
│   │       └── vault.yml          # Зашифрованный секрет (Ansible Vault)
│   └── prod/
│       ├── hosts.ini              # Хосты prod-окружения
│       └── group_vars/
│           ├── redis.yml          # Переменные prod
│           └── vault.yml          # Зашифрованный секрет (Ansible Vault)
└── roles/redis/
    ├── defaults/main.yml          # Значения по умолчанию (переопределяются)
    ├── handlers/main.yml          # Handler перезапуска сервиса
    ├── tasks/main.yml             # Задачи роли
    └── templates/redis.conf.j2    # Шаблон конфигурации Redis
```

## Запуск

Плейбук один и тот же для всех окружений — окружение выбирается через `-i`:

```bash
# Dev
ansible-playbook playbook.yaml -i inventories/dev

# Prod
ansible-playbook playbook.yaml -i inventories/prod
```

### Передача пароля Ansible Vault

Файлы `inventories/*/group_vars/vault.yml` зашифрованы. Пароль храните **отдельно**
от репозитория (например, в `.vault_pass.txt` — файл добавлен в `.gitignore`).

```bash
# Вариант 1: файл с паролем (не коммитить!)
ansible-playbook playbook.yaml -i inventories/dev --vault-password-file .vault_pass.txt

# Вариант 2: ввести пароль интерактивно
ansible-playbook playbook.yaml -i inventories/dev --ask-vault-pass
```

### Проверка идемпотентности

Второй запуск подряд не должен менять систему (`changed=0`):

```bash
ansible-playbook playbook.yaml -i inventories/dev
ansible-playbook playbook.yaml -i inventories/dev
```

### Проверка шифрованных значений

```bash
ansible-vault view inventories/prod/group_vars/vault.yml
ansible-vault edit inventories/prod/group_vars/vault.yml
```

## Переопределяемые переменные

| Переменная         | По умолчанию      | Описание                              |
| ------------------ | ----------------- | ------------------------------------- |
| `redis_port`       | `6379`            | Порт Redis                            |
| `redis_dir`        | `/var/lib/redis`  | Каталог для данных                    |
| `redis_databases`  | `16`              | Количество логических баз             |
| `redis_appendonly` | `"yes"`           | Включить AOF-персистентность          |
| `redis_password`   | `"changeme"`      | Пароль (`requirepass`). **В репозитории переопределяется через `vault_redis_password`** |

### Различия окружений

| Параметр            | dev          | prod             |
| ------------------- | ------------ | ---------------- |
| Порт                | 6380         | 6379             |
| `databases`         | 4            | 16               |
| Каталог данных      | `/mnt/redis` | `/var/lib/redis` |
| `appendonly`        | no           | yes              |

Переменные окружения заданы в `inventories/<env>/group_vars/redis.yml`,
секрет — в зашифрованном `inventories/<env>/group_vars/vault.yml`
(`redis_password: "{{ vault_redis_password }}"`).
