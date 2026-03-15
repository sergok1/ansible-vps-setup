# Ansible VPS Setup

Ansible playbook для начальной настройки безопасности VPS на Ubuntu/Debian. Автоматизирует создание пользователя, настройку SSH, файрвол UFW, защиту fail2ban, автообновления и email-уведомления о входе.

Протестировано на Ubuntu 20.04, 22.04, 24.04.

## Возможности

- **Управление пользователями** — создание админа с SSH-ключом и sudo без пароля
- **Защита SSH** — кастомный порт, отключение root-логина, парольной аутентификации, включение PAM
- **Файрвол UFW** — настраиваемые правила с rate limiting для SSH
- **Fail2ban** — защита от brute force с опциональными email-уведомлениями
- **Автообновления** — обновление системы по расписанию с автоперезагрузкой
- **msmtp** — отправка почты через SMTP (Gmail и др.)
- **Login Notify** — email-уведомления о каждом SSH-входе с GeoIP
- **tmux** — установка и настройка с поддержкой мыши и скролла
- **Hostname** — автоматическая настройка имени сервера

## Требования

- Ansible 2.10+
- Python 3.8+ на управляющей машине
- Целевая система: Ubuntu 20.04+ / Debian 11+ с доступом по SSH

### Установка коллекций

```bash
ansible-galaxy install -r requirements.yml
```

## Быстрый старт

### 1. Клонирование и настройка

```bash
git clone git@github.com:sergok1/ansible-vps-setup.git
cd ansible-vps-setup

# Копируем шаблоны
cp inventories/hosts.ini.example inventories/hosts.ini
cp group_vars/all/main.yml.example group_vars/all/main.yml
```

### 2. Настройка inventory

Отредактируйте `inventories/hosts.ini`:

```ini
[vps]
myserver ansible_host=192.168.1.100

[vps:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

> **Первый запуск:** подключайтесь как `root` (или существующий пользователь с sudo). После выполнения роли `users` можно сменить `ansible_user` на созданного админа.

### 3. Настройка переменных

Отредактируйте `group_vars/all/main.yml`:

```yaml
admin_user: "your_username"
ssh_public_key_path: "~/.ssh/id_ed25519.pub"
ssh_port: 2222

ufw_allowed_ports:
  - { port: "2222", proto: "tcp", comment: "SSH" }
  - { port: "80", proto: "tcp", comment: "HTTP" }
  - { port: "443", proto: "tcp", comment: "HTTPS" }

fail2ban_ignoreip:
  - "127.0.0.1/8"
  - "ВАШ_ДОМАШНИЙ_IP"
```

### 4. Запуск

```bash
# Проверка подключения
ansible all -m ping

# Тестовый запуск (покажет изменения без применения)
ansible-playbook site.yml --check --diff

# Выполнение всех ролей (кроме уведомлений)
ansible-playbook site.yml

# Если sudo требует пароль (до настройки NOPASSWD)
ansible-playbook site.yml -K
```

## Настройка уведомлений

Роли `msmtp` и `login_notify` по умолчанию **не запускаются** (тег `never`).

### 1. Создать vault с секретами

```bash
ansible-vault create group_vars/all/vault.yml
```

Добавьте:

```yaml
vault_smtp_email: "your-email@gmail.com"
vault_smtp_password: "xxxx-xxxx-xxxx-xxxx"  # App Password для Gmail
vault_notify_email: "alerts@gmail.com"
```

### 2. Настроить переменные в main.yml

```yaml
smtp_email: "{{ vault_smtp_email }}"
smtp_password: "{{ vault_smtp_password }}"
notify_email: "{{ vault_notify_email }}"
server_name: "MY-VPS"

fail2ban_email_enabled: true
fail2ban_destemail: "{{ notify_email }}"
fail2ban_sender: "{{ smtp_email }}"
```

### 3. Запустить

```bash
# Только уведомления
ansible-playbook site.yml --tags notifications --ask-vault-pass

# Всё вместе с уведомлениями
ansible-playbook site.yml --tags all,notifications --ask-vault-pass
```

## Шифрованный swap

Роль `encrypted_swap` по умолчанию **не запускается**. Создаёт swap-файл, шифрует его через `dm-crypt` с ключом из `/dev/urandom` — данные не переживают перезагрузку.

### 1. Включить в main.yml

```yaml
encrypted_swap_enabled: true
encrypted_swap_size: "512M"   # или "1G"
```

### 2. Запустить

```bash
ansible-playbook site.yml --tags swap --ask-vault-pass

# Или вместе со всем
ansible-playbook site.yml --tags all,swap --ask-vault-pass
```

## Приватность (логи и core dumps)

Роли `privacy_journal` и `privacy_coredump` по умолчанию **не запускаются**.

- **privacy_journal** — переводит systemd-journal в volatile (RAM). Логи живут только в оперативной памяти, после перезагрузки стираются. На диске не остаётся следов SSH-подключений, IP-адресов клиентов и т.д.
- **privacy_coredump** — отключает дампы памяти. При падении программы её память не сбрасывается на диск.

### 1. Включить в main.yml

```yaml
privacy_journal_enabled: true
privacy_coredump_enabled: true
```

### 2. Запустить

```bash
# Обе роли
ansible-playbook site.yml --tags privacy --ask-vault-pass

# По отдельности
ansible-playbook site.yml --tags privacy_journal --ask-vault-pass
ansible-playbook site.yml --tags privacy_coredump --ask-vault-pass
```

## Запуск отдельных ролей

```bash
ansible-playbook site.yml --tags common        # Обновление + hostname
ansible-playbook site.yml --tags users         # Создание пользователя
ansible-playbook site.yml --tags ssh           # Настройка SSH
ansible-playbook site.yml --tags ufw           # Файрвол
ansible-playbook site.yml --tags fail2ban      # Fail2ban
ansible-playbook site.yml --tags auto_update   # Автообновления
ansible-playbook site.yml --tags tmux          # tmux
ansible-playbook site.yml --tags swap          # Шифрованный swap
ansible-playbook site.yml --tags privacy       # Логи в RAM + отключение core dumps
ansible-playbook site.yml --tags privacy_journal  # Только логи в RAM
ansible-playbook site.yml --tags privacy_coredump # Только отключение core dumps

# Группы ролей
ansible-playbook site.yml --tags security      # SSH + UFW + fail2ban
ansible-playbook site.yml --tags notifications # msmtp + login_notify

# На конкретном хосте
ansible-playbook site.yml --limit myserver

# Пропустить роль
ansible-playbook site.yml --skip-tags users
```

## Структура проекта

```
ansible-vps-setup/
├── ansible.cfg
├── site.yml
├── requirements.yml
├── inventories/
│   ├── hosts.ini.example
│   └── hosts.ini            # gitignored
├── group_vars/
│   └── all/
│       ├── main.yml.example
│       ├── main.yml         # gitignored
│       ├── vault.yml.example
│       └── vault.yml        # gitignored, зашифрован
└── roles/
    ├── common/              # Обновление системы, hostname, пакеты
    ├── users/               # Пользователь, SSH-ключ, sudo NOPASSWD
    ├── ssh/                 # Безопасность SSH, UsePAM
    ├── ufw/                 # Правила файрвола
    ├── fail2ban/            # Защита от brute force
    ├── auto_update/         # Автообновления по расписанию
    ├── tmux/                # tmux с мышью и скроллом
    ├── encrypted_swap/      # Шифрованный swap (dm-crypt)
    ├── privacy_journal/     # Логи в RAM (volatile)
    ├── privacy_coredump/    # Отключение core dumps
    ├── msmtp/               # Отправка почты через SMTP
    └── login_notify/        # PAM-уведомления о SSH-входе
```

## Справочник параметров

### SSH

| Переменная | По умолчанию | Описание |
|---|---|---|
| `ssh_port` | 22 | Порт SSH |
| `ssh_permit_root_login` | no | Разрешить SSH для root |
| `ssh_password_authentication` | no | Парольная аутентификация |
| `ssh_max_auth_tries` | 3 | Макс. попыток аутентификации |
| `ssh_client_alive_interval` | 300 | Интервал проверки активности (сек) |
| `ssh_client_alive_count_max` | 2 | Макс. пропущенных проверок |

### UFW

| Переменная | По умолчанию | Описание |
|---|---|---|
| `ufw_default_incoming_policy` | deny | Политика входящих |
| `ufw_rate_limit_ssh` | true | Rate limiting для SSH |
| `ufw_allowed_ports` | [] | Список разрешённых портов |

### Fail2ban

| Переменная | По умолчанию | Описание |
|---|---|---|
| `fail2ban_maxretry` | 5 | Попыток до бана |
| `fail2ban_bantime` | -1 | Длительность бана (-1 = навсегда) |
| `fail2ban_ignoreip` | [127.0.0.1/8] | IP которые не банить |
| `fail2ban_email_enabled` | false | Email-уведомления о банах |

### Автообновления

| Переменная | По умолчанию | Описание |
|---|---|---|
| `auto_update_weekday` | 3 | День недели (0=вс, 3=ср) |
| `auto_update_hour` | 4 | Час запуска |
| `auto_update_reboot` | true | Автоперезагрузка |
| `auto_update_reboot_time` | 04:30 | Время перезагрузки |
| `auto_update_blacklist` | [] | Пакеты НЕ обновлять |

### tmux

| Переменная | По умолчанию | Описание |
|---|---|---|
| `tmux_mouse` | true | Поддержка мыши |
| `tmux_history_limit` | 10000 | Буфер прокрутки (строк) |
| `tmux_base_index` | 1 | Нумерация окон с 1 |
| `tmux_configure_root` | true | Настроить и для root |

### Шифрованный swap

| Переменная | По умолчанию | Описание |
|---|---|---|
| `encrypted_swap_enabled` | false | Включить роль |
| `encrypted_swap_size` | 512M | Размер swap-файла |
| `encrypted_swap_file` | /swapcrypt.img | Путь к файлу |
| `encrypted_swap_cipher` | aes-xts-plain64 | Алгоритм шифрования |
| `encrypted_swap_key_size` | 256 | Размер ключа (бит) |

### Приватность: логи

| Переменная | По умолчанию | Описание |
|---|---|---|
| `privacy_journal_enabled` | false | Включить роль |
| `journal_storage` | volatile | Хранение: volatile (RAM) или persistent (диск) |
| `journal_runtime_max_use` | 50M | Макс. размер логов в RAM |
| `journal_max_retention` | 1day | Время хранения |
| `journal_cleanup_persistent` | true | Удалить старые логи с диска |

### Приватность: core dumps

| Переменная | По умолчанию | Описание |
|---|---|---|
| `privacy_coredump_enabled` | false | Включить роль |
| `coredump_storage` | none | Хранение: none (отключить) или journal |
| `coredump_max_size` | 0 | Макс. размер дампа (0 = не сохранять) |
| `coredump_ulimit` | 0 | Лимит core в ulimits (0 = запретить) |

### Уведомления

| Переменная | По умолчанию | Описание |
|---|---|---|
| `smtp_host` | smtp.gmail.com | SMTP сервер |
| `smtp_port` | 587 | SMTP порт |
| `smtp_email` | — | Email отправителя (из vault) |
| `smtp_password` | — | Пароль SMTP (из vault) |
| `notify_email` | — | Email для уведомлений (из vault) |
| `server_name` | MY-VPS | Имя сервера в письмах |
| `login_notify_geoip` | true | Определять страну по IP |

## Важно

- Перед запуском убедитесь, что SSH-ключ настроен и работает
- Сначала протестируйте с `--check --diff`
- Держите запасной доступ (VNC/консоль провайдера) на случай блокировки
- Запомните новый SSH-порт для переподключения
- После первого запуска подключайтесь: `ssh -p ВАШ_ПОРТ ваш_пользователь@ip_сервера`

## Решение проблем

### Потерян SSH-доступ

1. Используйте VNC/консоль провайдера
2. Проверьте порт: `grep Port /etc/ssh/sshd_config`
3. Проверьте UFW: `ufw status`
4. Проверьте fail2ban: `fail2ban-client status sshd`

### Не приходят уведомления о входе

1. Проверьте `UsePAM yes` в sshd: `sudo sshd -T | grep usepam`
2. Проверьте PAM-строку: `grep login-notify /etc/pam.d/sshd`
3. Проверьте лог скрипта: `cat /var/log/login-notify.log`
4. Тест отправки: `echo "test" | sudo /usr/sbin/sendmail your@email.com`
5. Проверьте лог msmtp: `cat /var/log/msmtp.log`

### Ошибка sudo при Ansible

Если `Incorrect sudo password` — добавьте `vault_ansible_become_pass` в vault и раскомментируйте `ansible_become_pass` в inventory. Или настройте NOPASSWD через роль `users`.

Если `Timeout waiting for privilege escalation prompt` — у пользователя уже настроен NOPASSWD, уберите `ansible_become_pass` из inventory.

### Подробная отладка

```bash
ansible-playbook site.yml -vvv
ansible all -m ping -vvv
```

## Лицензия

MIT
