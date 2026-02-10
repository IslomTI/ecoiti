Модуль **В (C) — Автоматизация** (обычно это Ansible и Скрипты).
В этом модуле ты работаешь на машине администратора (например, `cli-cod` или `admin-cod`) и управляешь другими машинами (`srv`, `rtr`, `sw`) удаленно.

Вот шпаргалка, разбитая на **Ansible** (90% модуля) и **Python/Bash** (10% модуля).

---

# 🚀 FINAL CHEAT SHEET: MODULE C (Automation)

**Цель:** Написать Playbook для настройки серверов и скрипт для бэкапов/мониторинга.
**Инструменты:** Ansible, Python, Bash.

---

## 1. 🛠️ Подготовка (Setup)

Сначала установи Ansible и настрой SSH-доступ. Пароли вводить каждый раз нельзя.

```bash
# 1. Установка
apt-get update
apt-get install ansible sshpass git python3 -y

# 2. Генерация SSH-ключей (Жми Enter на все вопросы)
ssh-keygen -t rsa

# 3. Раскидываем ключи на ВСЕ хосты (чтобы заходить без пароля)
# (Пароль везде стандартный, например P@ssw0rd)
ssh-copy-id root@192.168.100.10  # srv1
ssh-copy-id root@192.168.100.11  # srv2
ssh-copy-id root@10.10.30.11     # sw1 (если там Linux)

```

---

## 2. 🅰️ Ansible: Структура

Создай папку для проекта (например, `/root/ansible`) и перейди в нее.
Внутри должно быть 2 файла: `ansible.cfg` и `hosts`.

### 📄 Файл 1: `ansible.cfg` (Настройки)

Этот файл отключает проверку ключей (чтобы не писать "yes" при подключении) и указывает инвентарь.

```ini
[defaults]
inventory = ./hosts
host_key_checking = False
remote_user = root
deprecation_warnings = False

```

### 📄 Файл 2: `hosts` (Инвентарь)

Здесь мы описываем группы серверов.

```ini
[webservers]
srv1-cod ansible_host=192.168.100.10

[dbservers]
srv2-cod ansible_host=192.168.100.11

[switches]
sw1-cod ansible_host=10.10.30.11

[all:vars]
# Если ssh-copy-id не сработал, можно задать пароль тут (но лучше ключи!)
ansible_ssh_pass=P@ssw0rd
ansible_python_interpreter=/usr/bin/python3

```

**Проверка связи:**

```bash
ansible all -m ping
# Должен ответить зеленым "pong"

```

---

## 3. 📜 Ansible: Playbook (Сценарий)

Создай файл `site.yml`. Это твой главный скрипт.
В заданиях обычно просят: установить Nginx, создать юзера, скопировать конфиг.

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Create index.html
      copy:
        dest: /var/www/html/index.html
        content: "<h1>Welcome to WorldSkills 2026</h1>"

- name: Configure DB Servers (Postgres)
  hosts: dbservers
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present

- name: Common settings (Users)
  hosts: all
  tasks:
    - name: Create Admin User
      user:
        name: ws_admin
        shell: /bin/bash
        groups: wheel
        password: "{{ 'P@ssw0rd' | password_hash('sha512') }}"

```

**Запуск плейбука:**

```bash
ansible-playbook site.yml

```

---

## 4. 🐍 Python: Скрипты (Мониторинг/Аудит)

Часто просят написать скрипт, который собирает инфу о системе (CPU, RAM) и кладет в файл.

Создай файл `audit.py`.

```python
#!/usr/bin/python3
import os
import subprocess
import datetime

# Функция для выполнения команд bash
def run_cmd(command):
    return subprocess.getoutput(command)

# Имя файла отчета
filename = f"/root/report_{datetime.date.today()}.txt"

# Сбор данных
hostname = run_cmd("hostname")
ip_addr = run_cmd("hostname -I")
cpu_load = run_cmd("uptime | awk -F'load average:' '{ print $2 }'")
disk_space = run_cmd("df -h / | tail -1 | awk '{print $4}'") # Free space

# Формирование текста
report = f"""
=== SYSTEM REPORT ===
Date: {datetime.datetime.now()}
Hostname: {hostname}
IP Address: {ip_addr}
CPU Load: {cpu_load}
Free Disk: {disk_space}
=====================
"""

# Запись в файл
with open(filename, "w") as f:
    f.write(report)

print(f"Report saved to {filename}")

```

**Запуск:** `python3 audit.py`

---

## 5. 🐚 Bash: Скрипт Бэкапа (Backup)

Задание: "Делать бэкап папки /etc каждый день и удалять старые бэкапы (старше 7 дней)".

Создай файл `backup.sh`.

```bash
#!/bin/bash

# Переменные
SOURCE_DIR="/etc"
BACKUP_DIR="/var/backups/config_backups"
DATE=$(date +%F) # Формат ГГГГ-ММ-ДД
FILENAME="backup_$DATE.tar.gz"

# 1. Создаем папку, если нет
mkdir -p $BACKUP_DIR

# 2. Архивируем
tar -czf $BACKUP_DIR/$FILENAME $SOURCE_DIR

# 3. Проверка результата
if [ $? -eq 0 ]; then
    echo "Backup success: $FILENAME"
else
    echo "Backup failed!"
    exit 1
fi

# 4. Удаление старых файлов (старше 7 дней)
find $BACKUP_DIR -name "backup_*.tar.gz" -mtime +7 -delete

```

**Сделать исполняемым:** `chmod +x backup.sh`

---

## 6. ⏰ Cron (Планировщик)

Чтобы скрипты запускались сами (например, каждые 5 минут или каждый день).

1. Открой редактор cron:
```bash
crontab -e

```


2. Добавь строки вниз:
```bash
# Запуск Ansible каждые 30 минут (для поддержания конфигурации)
*/30 * * * * ansible-playbook /root/ansible/site.yml >> /var/log/ansible.log 2>&1

# Бэкап каждый день в 03:00 ночи
0 3 * * * /root/backup.sh

```



---

## ⚡ Самые частые модули Ansible (Для запоминания)

1. **`apt`** / **`package`**: Установка программ.
* `name: nginx`, `state: present`.


2. **`service`**: Управление службами.
* `name: sshd`, `state: restarted`, `enabled: yes`.


3. **`copy`**: Копирование файла с твоего компа на сервер.
* `src: file.conf`, `dest: /etc/file.conf`.


4. **`template`**: То же, что copy, но можно менять переменные внутри файла (Jinja2).
5. **`file`**: Создание папок, смена прав.
* `path: /opt/data`, `state: directory`, `mode: '0755'`.


6. **`user`**: Создание пользователей.
7. **`lineinfile`**: Изменение одной строчки в конфиге (например, разрешить SSH root).
* `path: /etc/ssh/sshd_config`, `regexp: '^PermitRootLogin'`, `line: 'PermitRootLogin yes'`.