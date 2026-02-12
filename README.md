```markdown
# Развёртывание Drupal с помощью Ansible (Docker + Ubuntu 22.04)

## 1. Подготовка управляемого хоста

В связи с невозможностью использования Vagrant, управляемый хост был развёрнут в Docker-контейнере на базе **Ubuntu 22.04**.

### 1.1. Запуск контейнера и настройка SSH

```bash
# Запуск контейнера с пробросом портов
docker run -d --name drupal-host -p 2222:22 -p 8080:80 ubuntu:22.04 sleep infinity

# Установка SSH-сервера и настройка доступа
docker exec drupal-host bash -c "
apt update && \
apt install -y openssh-server && \
mkdir /var/run/sshd && \
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config && \
service ssh start
"

# Копирование публичного SSH-ключа
docker cp ~/.ssh/id_rsa.pub drupal-host:/root/.ssh/authorized_keys
docker exec drupal-host chown root:root /root/.ssh/authorized_keys
docker exec drupal-host chmod 600 /root/.ssh/authorized_keys
```

В результате получен управляемый хост с адресом `127.0.0.1:2222`, пользователь `root`.

---

## 2. Настройка управляющего узла (локальная машина)

### 2.1. Структура проекта

```
drupal-ansible/
├── ansible.cfg
├── inventory
├── vars.yml
├── playbook.yml
├── templates/
│   ├── nginx-drupal.conf.j2
│   └── settings.php.j2
├── drupal_deploy.log
└── README.md
```

### 2.2. Содержимое конфигурационных файлов

**`ansible.cfg`**
```ini
[defaults]
host_key_checking = False
inventory = inventory
```

**`inventory`**
```ini
drupal ansible_host=127.0.0.1 ansible_port=2222 ansible_user=root
```

**`vars.yml`**
```yaml
drupal_domain: localhost
drupal_root: /var/www/drupal
db_name: drupal
db_user: drupal
db_password: drupal123
db_host: localhost
```

**Шаблоны конфигурации**

`templates/nginx-drupal.conf.j2`:
```nginx
server {
    listen 80;
    server_name {{ drupal_domain }};
    root {{ drupal_root }}/web;

    index index.php;

    location / {
        try_files $uri /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

`templates/settings.php.j2`:
```php
<?php
$databases['default']['default'] = [
  'database' => '{{ db_name }}',
  'username' => '{{ db_user }}',
  'password' => '{{ db_password }}',
  'prefix' => '',
  'host' => '{{ db_host }}',
  'port' => '5432',
  'namespace' => 'Drupal\\pgsql\\Driver\\Database\\pgsql',
  'driver' => 'pgsql',
];

$settings['config_sync_directory'] = '{{ drupal_root }}/config/sync';
$settings['file_private_path'] = '{{ drupal_root }}/private';
```

### 2.3. Итоговый плейбук `playbook.yml`

Плейбук написан **без использования сторонних ролей** (из-за сетевых ограничений и несовместимости). Он выполняет:

- Установку и настройку PostgreSQL 14, Nginx, PHP 8.1‑FPM, Composer.
- Создание базы данных и пользователя БД.
- Развёртывание Drupal 10 через Composer.
- Автоматическую установку Drupal с помощью Drush.
- Настройку прав доступа и конфигурационных файлов.

Полное содержимое `playbook.yml` доступно в [репозитории](#).

---

## 3. Запуск плейбука

```bash
cd ~/drupal-ansible
ansible-playbook -i inventory playbook.yml -v > drupal_deploy.log 2>&1
```

Лог выполнения (объёмный, более 1000 строк) сохранён в файле `drupal_deploy.log` и прилагается к проекту.

---

## 4. Проверка работоспособности

- Drupal доступен по адресу: **http://localhost:8080**
- Учётные данные для входа: **admin / admin**
- Во время установки отображается предупреждение о версии PHP (используется PHP 8.1, а рекомендуется 8.3). Это **не является критическим** — Drupal 10 стабильно работает на PHP 8.1, задание выполнено в полном объёме.

---

## 5. Состав сдаваемых файлов

Все файлы находятся в данном репозитории:

| Файл | Назначение |
|------|------------|
| `playbook.yml` | Основной сценарий Ansible |
| `vars.yml` | Переменные для плейбука |
| `inventory` | Инвентарный файл с адресом хоста |
| `ansible.cfg` | Конфигурация Ansible |
| `templates/nginx-drupal.conf.j2` | Шаблон конфигурации Nginx |
| `templates/settings.php.j2` | Шаблон `settings.php` Drupal |
| `drupal_deploy.log` | Лог выполнения плейбука |
| `README.md` | Данный файл с описанием |

---

