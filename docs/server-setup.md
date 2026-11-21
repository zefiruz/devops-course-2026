# Конфигурация виртуальной машины devops-vm

## 1. Параметры машины
- **ОС:** Ubuntu Server 24.04 LTS (64-bit)
- **Оперативная память:** 2048 МБ
- **Процессор:** 2 ядра
- **Дисковый накопитель:** 25 ГБ (динамически расширяемый)

## 2. Сетевые интерфейсы
- **Адаптер 1 (`enp0s3`):** Тип `NAT`, IP `10.0.2.15/24`. Назначение: доступ виртуальной машины в Интернет и подключение по SSH с хоста через проброс портов.
- **Адаптер 2 (`enp0s8`):** Тип `Host-only`, IP `192.168.56.101/24`. Назначение: прямой сетевой доступ с хостовой системы к сервисам ВМ без проброса портов.
- **Локальное доменное имя:**
  - На хосте в `hosts`: `192.168.56.101 devops.local`
  - На сервере (FQDN в `/etc/hosts`): `127.0.1.1 devops-vm.devops.local devops-vm`

## 3. Правило проброса портов
- **Адаптер 1 (NAT):**
  - Наименование: `ssh`
  - Протокол: `TCP`
  - Адрес хоста: `127.0.0.1`
  - Порт хоста: `2222`
  - Порт гостя: `2222` (на этапе первичной настройки — `22`)

## 4. Учётные записи
- `student` (UID 1000) — первичная учётная запись, группы: `student, sudo`.
- `devops` (UID 1001) — основная рабочая учётная запись администратора, группы: `devops, sudo`. Аутентификация: строго по SSH-ключу `Ed25519` (`~/.ssh/devops_vm`, на сервере `~/.ssh/authorized_keys` с правами `600`, каталог `~/.ssh` с правами `700`).

## 5. Служба SSH
- **Расположение файла конфигурации:** `/etc/ssh/sshd_config.d/99-hardening.conf` (в `50-cloud-init.conf` закомментирован `PasswordAuthentication yes`).
- **Режим запуска:** отключён `ssh.socket`, активирован `ssh.service`.
- **Установленные директивы:**
  - `Port 2222`
  - `PermitRootLogin no`
  - `PasswordAuthentication no`
  - `PubkeyAuthentication yes`
  - `PermitEmptyPasswords no`
  - `MaxAuthTries 3`
  - `LoginGraceTime 30`
  - `AllowUsers devops`
  - `X11Forwarding no`
  - `ClientAliveInterval 300`
  - `ClientAliveCountMax 2`

## 6. Правила межсетевого экрана (UFW)
- **Политики по умолчанию:** входящий трафик — `deny (incoming)`, исходящий трафик — `allow (outgoing)`.
- **Уровень логирования:** `medium`.
- **Разрешающие правила:**
  - `2222/tcp` — `LIMIT IN` (SSH rate-limited)
  - `80/tcp` — `ALLOW IN` (HTTP)
  - `443/tcp` — `ALLOW IN` (HTTPS)

## 7. Снимки состояния
1. `01-clean-install` — после чистой установки ОС и обновления пакетов.
2. `02-keys-configured` — после создания пользователя `devops` и настройки входа по SSH-ключу через `ssh devops`.
3. `03-ssh-hardened` — после применения параметров безопасности `sshd` и перевода службы на порт `2222`.

## 8. Веб-сервер
- **Устанавливаемый пакет:** `nginx` (`sudo apt install -y nginx`)
- **Путь к конфигурации ресурса:** `/etc/nginx/sites-available/devops-site` (активирован символической ссылкой `/etc/nginx/sites-enabled/devops-site`, стандартный `/etc/nginx/sites-enabled/default` удалён).
- **Каталог ресурса:** `/var/www/devops-site`
  - **Владелец и группа:** `devops:devops`
  - **Права доступа:** каталог — `755` (`drwxr-xr-x`), файлы — `644` (`-rw-r--r--`).
- **TLS-сертификат и закрытый ключ:**
  - **Путь к сертификату:** `/etc/ssl/certs/devops.crt` (владелец `root:root`, права `644`).
  - **Путь к закрытому ключу:** `/etc/ssl/private/devops.key` (владелец `root:root`, права `600`).
  - **Срок действия:** 365 дней.
  - **Команда формирования сертификата:**
    ```bash
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/ssl/private/devops.key \
      -out /etc/ssl/certs/devops.crt \
      -subj "/CN=devops.local" \
      -addext "subjectAltName=DNS:devops.local"
    ```

## 10. Ansible