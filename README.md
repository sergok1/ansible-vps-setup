# Ansible VPS Setup

Ansible playbook для начальной настройки безопасности VPS. Автоматизирует создание пользователя, настройку SSH, конфигурацию файрвола UFW и защиту fail2ban.

## Возможности

- **Управление пользователями**: Создание админа с sudo и SSH-ключами
- **Защита SSH**: Кастомный порт, отключение root-логина и парольной аутентификации
- **Файрвол UFW**: Настраиваемые правила с rate limiting для SSH
- **Fail2ban**: Защита от brute force атак

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
git clone https://github.com/YOUR_USERNAME/ansible-vps-setup.git
cd ansible-vps-setup

# Копируем примеры файлов
cp inventories/hosts.ini.example inventories/hosts.ini
cp group_vars/all.yml.example group_vars/all.yml
```

### 2. Редактирование inventory

Отредактируйте `inventories/hosts.ini`, указав IP ваших серверов:

```ini
[vps]
server1 ansible_host=192.168.1.100
server2 ansible_host=192.168.1.101
server3 ansible_host=10.0.0.50

[vps:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### 3. Настройка переменных

Отредактируйте `group_vars/all.yml`:

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

# Доверенные IP для fail2ban
fail2ban_ignoreip:
  - "127.0.0.1/8"
  - "ВАШ_ДОМАШНИЙ_IP"
```

### 4. Запуск playbook

```bash
# Проверка подключения
ansible all -m ping

# Тестовый запуск (без изменений)
ansible-playbook site.yml --check

# Выполнение
ansible-playbook site.yml
```

## Примеры использования

### Запуск отдельных ролей

```bash
# Только файрвол
ansible-playbook site.yml --tags ufw

# SSH и fail2ban
ansible-playbook site.yml --tags "ssh,fail2ban"

# Пропустить создание пользователя
ansible-playbook site.yml --skip-tags users
```

### Запуск на конкретных хостах

```bash
# Один хост
ansible-playbook site.yml --limit server1

# Несколько хостов
ansible-playbook site.yml --limit "server1,server2"
```

### Использование vault для секретов

```bash
# Создать зашифрованный файл
ansible-vault create group_vars/vault.yml

# Редактировать зашифрованный файл
ansible-vault edit group_vars/vault.yml

# Запуск с vault
ansible-playbook site.yml --ask-vault-pass
```

## Структура проекта

```
ansible-vps-setup/
├── ansible.cfg              # Конфигурация Ansible
├── site.yml                 # Главный playbook
├── inventories/
│   ├── hosts.ini.example    # Шаблон inventory
│   └── hosts.ini            # Ваш inventory (игнорируется git)
├── group_vars/
│   ├── all.yml.example      # Шаблон переменных
│   └── all.yml              # Ваши переменные (игнорируется git)
└── roles/
    ├── common/              # Обновление системы, пакеты
    ├── users/               # Создание пользователей, SSH ключи
    ├── ssh/                 # Настройка безопасности SSH
    ├── ufw/                 # Правила файрвола
    └── fail2ban/            # Защита от brute force
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
| `fail2ban_bantime` | 86400 | Длительность бана (секунды) |
| `fail2ban_ignoreip` | [127.0.0.1/8] | IP которые никогда не банить |

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

## Лицензия

MIT
