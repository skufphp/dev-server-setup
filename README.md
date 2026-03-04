# dev-server-setup

Ansible-проект для автоматической раскатки **DEV**-серверов по стратегии **Black Friday Rolling Migration**.

Стек: **Docker Swarm + Dokploy + Ansible + Tailscale**

> **Это dev-окружение.** Используется для тестирования и репетиции миграций перед применением в production. Для prod-окружения используйте отдельный проект [prod-server-setup](../prod-server-setup).

---

## Оглавление

- [Структура проекта](#структура-проекта)
- [Требования](#требования)
- [Быстрый старт](#быстрый-старт)
- [Inventory — настройка серверов](#inventory--настройка-серверов)
- [Переменные](#переменные)
- [Роли](#роли)
- [Плейбуки](#плейбуки)
- [Сценарии использования](#сценарии-использования)
- [Чеклист миграции](#чеклист-миграции)
- [Troubleshooting](#troubleshooting)

---

## Структура проекта

```
dev-server-setup/
├── ansible.cfg                  # Конфигурация Ansible
├── inventory/
│   └── dev.ini                  # Inventory серверов (MagicDNS имена)
├── group_vars/
│   └── all.yml                  # Общие переменные
├── roles/
│   ├── common/                  # Базовая настройка ОС
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml
│   ├── docker/                  # Установка Docker
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml
│   ├── tailscale/               # Установка и настройка Tailscale VPN
│   │   └── tasks/main.yml
│   ├── firewall/                # UFW + fail2ban
│   │   └── tasks/main.yml
│   └── swarm/                   # Инициализация Docker Swarm
│       └── tasks/main.yml
├── playbooks/
│   ├── setup.yml                # Полная раскатка сервера
│   ├── ping.yml                 # Проверка доступности
│   └── migrate-db.yml           # Миграция PostgreSQL
├── STRATEGY.md                  # Описание стратегии миграции
└── DETAILS.md                   # Детали по Tailscale и ACL
```

---

## Требования

### На вашем MacBook (управляющая машина)

1. **Ansible** ≥ 2.14:
   ```bash
   brew install ansible
   ```

2. **Tailscale** — установлен и авторизован:
   ```bash
   brew install tailscale
   ```

3. **SSH-ключ** — добавлен на все целевые серверы.

### На целевых серверах (VPS)

- **Ubuntu** 22.04 / 24.04
- SSH-доступ по ключу для пользователя `ubuntu`
- Доступ к `sudo` без пароля

### Tailscale

- Аккаунт на [tailscale.com](https://tailscale.com)
- **MagicDNS включён** (Admin Console → DNS)
- **Auth Key** сгенерирован (Settings → Keys → Generate auth key)
- MacBook помечен тегом `tag:infra`
- Серверы помечены тегом `tag:dev`

---

## Быстрый старт

### 1. Клонировать репозиторий

```bash
git clone <repo-url> dev-server-setup
cd dev-server-setup
```

### 2. Настроить inventory

Отредактируйте `inventory/dev.ini` — укажите MagicDNS-имена или Tailscale IP ваших серверов:

```ini
[managers]
dev-manager-1

[workers]
dev-worker-1
```

### 3. Задать Tailscale Auth Key

```bash
export TAILSCALE_AUTH_KEY="tskey-auth-xxxxx"
```

### 4. Проверить доступность серверов

```bash
ansible-playbook playbooks/ping.yml
```

### 5. Раскатать серверы

```bash
# Все серверы
ansible-playbook playbooks/setup.yml

# Только один новый сервер
ansible-playbook playbooks/setup.yml --limit dev-manager-1
```

---

## Inventory — настройка серверов

Файл: `inventory/dev.ini`

Серверы разделены на группы:

| Группа | Описание |
|--------|----------|
| `managers` | Manager-ноды dev-кластера |
| `workers` | Worker-ноды dev-кластера |

> **Важно:** используйте MagicDNS-имена (требуется включённый MagicDNS в Tailscale). Если MagicDNS не включён — используйте Tailscale IP (`100.x.x.x`).

---

## Переменные

Файл: `group_vars/all.yml`

| Переменная | Описание | Значение по умолчанию |
|------------|----------|-----------------------|
| `tailscale_auth_key` | Auth key для Tailscale | из env `TAILSCALE_AUTH_KEY` |
| `docker_edition` | Редакция Docker | `ce` |
| `common_packages` | Список системных пакетов | см. файл |
| `ssh_port` | Порт SSH | `22` |
| `ssh_permit_root_login` | Разрешить root login | `no` |
| `ssh_password_authentication` | Аутентификация по паролю | `no` |
| `ufw_default_incoming` | Политика UFW (входящие) | `deny` |
| `swarm_ports` | Порты Docker Swarm | 2377, 7946, 4789 |

---

## Роли

### `common` — Базовая настройка

- Обновление системы и установка пакетов
- Настройка SSH (запрет root, запрет паролей)
- Настройка часового пояса (UTC)
- Автоматические обновления безопасности
- Создание swap (2 GB)

### `docker` — Docker Engine

- Удаление старых версий Docker
- Установка Docker CE из официального репозитория
- Docker Compose plugin
- Настройка log rotation
- Добавление пользователя в группу `docker`

### `tailscale` — Mesh VPN

- Установка Tailscale из официального репозитория
- Авторизация ноды по auth key
- Получение и сохранение Tailscale IP как факта (`tailscale_ipv4`)

### `firewall` — UFW + fail2ban

- Политика по умолчанию: deny incoming, allow outgoing
- Разрешены: SSH, HTTP/HTTPS, Tailscale UDP
- Swarm-порты разрешены только через интерфейс `tailscale0`
- fail2ban для защиты от брутфорса

### `swarm` — Docker Swarm

- Инициализация Swarm на manager-нодах с `--advertise-addr` = Tailscale IP
- Автоматическое получение join-токенов
- Присоединение worker-нод к кластеру

---

## Плейбуки

### `playbooks/setup.yml` — Полная раскатка

Применяет все роли последовательно: common → docker → tailscale → firewall → swarm.

```bash
# Все серверы
ansible-playbook playbooks/setup.yml

# Один конкретный сервер
ansible-playbook playbooks/setup.yml --limit dev-manager-1

# Dry-run (проверка без изменений)
ansible-playbook playbooks/setup.yml --check
```

### `playbooks/ping.yml` — Проверка доступности

```bash
ansible-playbook playbooks/ping.yml
```

### `playbooks/migrate-db.yml` — Миграция PostgreSQL

```bash
ansible-playbook playbooks/migrate-db.yml \
  -e "source_host=dev-manager-1" \
  -e "target_host=dev-manager-2" \
  -e "db_name=mydb" \
  -e "db_user=postgres"
```

---

## Сценарии использования

### Сценарий 1: Первоначальная настройка (один VPS)

```bash
# 1. Добавить сервер в inventory/dev.ini
# 2. Экспортировать Tailscale auth key
export TAILSCALE_AUTH_KEY="tskey-auth-xxxxx"

# 3. Раскатать
ansible-playbook playbooks/setup.yml --limit dev-manager-1
```

После этого на сервере будут: Docker, Tailscale, UFW, fail2ban, Docker Swarm (single-node manager).

Далее вручную установите **Dokploy** на manager-ноде и настройте проекты.

### Сценарий 2: Black Friday — добавление нового сервера

```bash
# 1. Купить новый VPS на распродаже
# 2. Добавить в inventory (например, dev-manager-2)
# 3. Раскатать
ansible-playbook playbooks/setup.yml --limit dev-manager-2

# 4. Promote до manager (на старом сервере)
ssh ubuntu@dev-manager-1 "docker node promote dev-manager-2"

# 5. Мигрировать БД
ansible-playbook playbooks/migrate-db.yml \
  -e "source_host=dev-manager-1" \
  -e "target_host=dev-manager-2" \
  -e "db_name=mydb" \
  -e "db_user=postgres"

# 6. Установить Dokploy на новом сервере, настроить проекты
# 7. Переключить DNS A-записи на IP нового сервера
# 8. Наблюдать 1–2 недели
# 9. Не продлевать старый сервер ✓
```

### Сценарий 3: Масштабирование (добавление worker-ноды)

```bash
# 1. Добавить новый сервер в секцию [workers] в inventory
# 2. Раскатать
ansible-playbook playbooks/setup.yml --limit dev-worker-2
```

Worker автоматически присоединится к Swarm-кластеру.

---

## Чеклист миграции

По стратегии **Black Friday Rolling Migration** (overlap period ~6 месяцев):

- [ ] Купить новый VPS на распродаже
- [ ] Добавить сервер в `inventory/dev.ini`
- [ ] Запустить `ansible-playbook playbooks/setup.yml --limit <новый_сервер>`
- [ ] Проверить, что сервер появился в Tailscale (`tailscale status`)
- [ ] Проверить, что сервер в Swarm (`docker node ls`)
- [ ] Promote до manager: `docker node promote <новый_сервер>`
- [ ] Сделать `pg_dump` и восстановить БД на новом сервере
- [ ] Установить Dokploy, настроить проекты и env-переменные
- [ ] Переключить DNS A-записи
- [ ] Наблюдать 1–2 недели
- [ ] Не продлевать старый сервер ✓

---

## Troubleshooting

### Ansible не может подключиться к серверу

```bash
# Проверить, что Tailscale работает
tailscale status

# Проверить SSH-доступ напрямую
ssh ubuntu@dev-manager-1

# Проверить ping через Ansible
ansible dev-manager-1 -m ping
```

### Tailscale не авторизуется

- Убедитесь, что `TAILSCALE_AUTH_KEY` задан и не истёк
- Сгенерируйте новый ключ: Tailscale Admin → Settings → Keys

### Swarm init падает

- Убедитесь, что Tailscale IP доступен: `tailscale ip -4`
- Проверьте, что порт 2377 открыт через tailscale0

### UFW блокирует Swarm-трафик

```bash
# Проверить правила
sudo ufw status verbose

# Swarm-порты должны быть разрешены на tailscale0
```

### Повторный запуск плейбука

Все роли **идемпотентны** — повторный запуск безопасен и не сломает существующую конфигурацию.

```bash
ansible-playbook playbooks/setup.yml
```

---

## Полезные команды

```bash
# Статус всех нод Swarm
ssh ubuntu@dev-manager-1 "docker node ls"

# Статус Tailscale на всех серверах
ansible all -a "tailscale status"

# Проверить UFW на всех серверах
ansible all -a "ufw status" --become

# Ad-hoc команда на группу серверов
ansible all -a "uptime"

# Обновить только Docker
ansible-playbook playbooks/setup.yml --tags docker
```

---

## Ссылки

- [STRATEGY.md](STRATEGY.md) — полное описание стратегии Black Friday Rolling Migration
- [DETAILS.md](DETAILS.md) — детали настройки Tailscale, ACL и MagicDNS
