# Ansible VPS Setup

Ansible playbook для начальной настройки безопасности VPS. Автоматизирует создание пользователя, настройку SSH, конфигурацию файрвола UFW, защиту fail2ban и настройку уведомлений.

## Возможности

- **Управление пользователями**: Создание админа с sudo и SSH-ключами
- **Защита SSH**: Кастомный порт, отключение root-логина и парольной аутентификации
- **Файрвол UFW**: Настраиваемые правила с rate limiting для SSH
- **Fail2ban**: Защита от brute force атак с опциональными email-уведомлениями
- **msmtp**: Настройка отправки почты через SMTP (Gmail и др.)
- **Login Notify**: Уведомления о любом входе в систему через PAM

## Требования

- Ansible 2.10+
- Python 3.8+ на управляющей машине
- Целевая система: Debian/Ubuntu VPS с root-доступом

### Ansible коллекции

```bash
ansible-galaxy collection install community.general ansible.posix
```

## Быстрый старт

### 1. Клонирование и настройка

```bash
git clone git@github.com:sergok1/ansible-vps-setup.git
cd ansible-vps-setup

# Копируем примеры файлов
cp inventories/hosts.ini.example inventories/hosts.ini
cp group_vars/all/main.yml.example group_vars/all/main.yml

# Создаём vault для секретов (если нужны уведомления)
ansible-vault create group_vars/all/vault.yml
```

### 2. Редактирование inventory

Отредактируйте `inventories/hosts.ini`, указав IP ваших серверов:

```ini
[vps]
# ALIAS — произвольное имя для хоста (отображается в выводе Ansible)
myserver ansible_host=192.168.1.100
web-server ansible_host=192.168.1.101

[vps:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### 3. Настройка переменных

Отредактируйте `group_vars/all/main.yml`:

```yaml
# Обязательные параметры
admin_user: "your_username"
ssh_public_key_path: "~/.ssh/id_ed25519.pub"
ssh_port: 2222  # Кастомный SSH порт

# Порты UFW
ufw_allowed_ports:
  - { port: "2222", proto: "tcp", comment: "SSH" }
  - { port: "80", proto: "tcp", comment: "HTTP" }
  - { port: "443", proto: "tcp", comment: "HTTPS" }

# Доверенные IP для fail2ban (не попадут в бан)
fail2ban_ignoreip:
  - "127.0.0.1/8"
  - "ВАШ_ДОМАШНИЙ_IP"
```

### 4. Запуск playbook

```bash
# Проверка подключения
ansible all -m ping

# Тестовый запуск — покажет ЧТО изменится, но НЕ применит изменения
ansible-playbook site.yml --check --diff

# Выполнение
ansible-playbook site.yml

# С sudo паролем
ansible-playbook site.yml -K
```

## Настройка уведомлений

Роли уведомлений (`msmtp`, `login_notify`) по умолчанию **не запускаются**. Для их включения нужно:

### 1. Создать vault с секретами

```bash
ansible-vault create group_vars/vault.yml
```

Добавьте:
```yaml
vault_smtp_email: "your-email@gmail.com"
vault_smtp_password: "xxxx-xxxx-xxxx-xxxx"  # App Password для Gmail
vault_notify_email: "alerts@gmail.com"
```

### 2. Настроить переменные в main.yml

```yaml
# Подключение секретов из vault
smtp_email: "{{ vault_smtp_email }}"
smtp_password: "{{ vault_smtp_password }}"
notify_email: "{{ vault_notify_email }}"

# Имя сервера в уведомлениях
server_name: "MY-VPS"

# Включить email уведомления fail2ban
fail2ban_email_enabled: true
fail2ban_destemail: "{{ notify_email }}"
fail2ban_sender: "{{ smtp_email }}"
```

### 3. Запустить роли уведомлений

```bash
# Только msmtp
ansible-playbook site.yml --tags msmtp --ask-vault-pass -K

# Только уведомления о входе
ansible-playbook site.yml --tags login_notify --ask-vault-pass -K

# Обе роли
ansible-playbook site.yml --tags notifications --ask-vault-pass -K

# Всё вместе (безопасность + уведомления)
ansible-playbook site.yml --tags "security,notifications" --ask-vault-pass -K
```

## Примеры использования

### Запуск отдельных ролей

```bash
# Базовые роли
ansible-playbook site.yml --tags common        # Обновление системы
ansible-playbook site.yml --tags users         # Создание пользователя
ansible-playbook site.yml --tags ssh           # Настройка SSH
ansible-playbook site.yml --tags ufw           # Файрвол
ansible-playbook site.yml --tags fail2ban      # Fail2ban

# Группы ролей
ansible-playbook site.yml --tags security      # SSH + UFW + fail2ban
ansible-playbook site.yml --tags notifications # msmtp + login_notify

# Пропустить роль
ansible-playbook site.yml --skip-tags users
```

### Запуск на конкретных хостах

```bash
ansible-playbook site.yml --limit myserver
ansible-playbook site.yml --limit "server1,server2"
```

### Подключение по паролю

```bash
ansible-playbook site.yml --ask-pass           # SSH пароль
ansible-playbook site.yml --ask-become-pass    # sudo пароль
ansible-playbook site.yml -k -K                # оба сразу
```

### Использование vault

```bash
# С запросом пароля vault
ansible-playbook site.yml --ask-vault-pass

# С файлом пароля
echo "ваш_vault_пароль" > .vault_pass
chmod 600 .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass
```

## Структура проекта

```
ansible-vps-setup/
├── ansible.cfg              # Конфигурация Ansible
├── site.yml                 # Главный playbook
├── inventories/
│   ├── hosts.ini.example    # Шаблон inventory
│   └── hosts.ini            # Ваш inventory (gitignored)
├── group_vars/
│   └── all/                 # Переменные для всех хостов
│       ├── main.yml.example # Шаблон переменных
│       ├── main.yml         # Ваши переменные (gitignored)
│       ├── vault.yml.example# Шаблон секретов
│       └── vault.yml        # Секреты (gitignored, зашифрован)
└── roles/
    ├── common/              # Обновление системы, пакеты
    ├── users/               # Создание пользователей, SSH ключи
    ├── ssh/                 # Настройка безопасности SSH
    ├── ufw/                 # Правила файрвола
    ├── fail2ban/            # Защита от brute force
    ├── msmtp/               # Настройка отправки почты
    └── login_notify/        # PAM уведомления о входе
```

## Справочник параметров

### Настройки SSH

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `ssh_port` | 22 | Порт SSH |
| `ssh_permit_root_login` | no | Разрешить SSH для root |
| `ssh_password_authentication` | no | Разрешить парольную аутентификацию |
| `ssh_max_auth_tries` | 3 | Макс. попыток аутентификации |

### Настройки UFW

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `ufw_default_incoming_policy` | deny | Политика входящих подключений |
| `ufw_rate_limit_ssh` | true | Rate limiting для SSH |
| `ufw_allowed_ports` | [] | Список разрешённых портов |

### Настройки Fail2ban

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `fail2ban_maxretry` | 5 | Макс. неудачных попыток до бана |
| `fail2ban_bantime` | -1 | Длительность бана (-1 = постоянный) |
| `fail2ban_ignoreip` | [127.0.0.1/8] | IP которые никогда не банить |
| `fail2ban_email_enabled` | false | Включить email уведомления |

### Настройки уведомлений

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `smtp_host` | smtp.gmail.com | SMTP сервер |
| `smtp_port` | 587 | SMTP порт |
| `smtp_email` | — | Email отправителя (из vault) |
| `smtp_password` | — | Пароль SMTP (из vault) |
| `notify_email` | — | Email для уведомлений (из vault) |
| `server_name` | MY-VPS | Имя сервера в письмах |
| `login_notify_geoip` | true | Определять страну по IP |

## Важно о безопасности

⚠️ **Перед запуском:**

1. Убедитесь, что SSH-ключ настроен
2. Сначала протестируйте на тестовом сервере
3. Держите запасной способ доступа (VNC/консоль провайдера)
4. Запомните новый SSH порт для переподключения

**После выполнения:**

```bash
# Подключение с новыми настройками
ssh -p ВАШ_SSH_ПОРТ ваш_пользователь@ip_сервера
```

## Решение проблем

### Потерян SSH доступ

1. Используйте консоль/VNC вашего VPS провайдера
2. Проверьте `/etc/ssh/sshd_config` на предмет порта
3. Проверьте правила UFW: `ufw status`
4. Проверьте fail2ban: `fail2ban-client status`

### Проблемы с подключением Ansible

```bash
# Подробный вывод
ansible-playbook site.yml -vvv

# Проверка связи
ansible all -m ping -vvv
```

### Не приходят уведомления

```bash
# Проверить лог msmtp
tail -f /var/log/msmtp.log

# Тест отправки письма
echo "Test" | mailx -s "Test" your@email.com

# Проверить PAM
sudo tail -f /var/log/auth.log
```

## Лицензия

MIT
