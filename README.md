# Домашнее задание к занятию 4 «Работа с roles» - Юрочкин В.А.

## Описание

В рамках домашнего задания исходный Ansible playbook был переработан на использование отдельных roles.

Цель работы — разделить установку и настройку компонентов на независимые роли и подключить их в основном playbook через `requirements.yml`.

В результате подготовлены три публичных репозитория:

- роль для установки и настройки Vector;
- роль для установки и настройки LightHouse;
- основной playbook, использующий роли ClickHouse, Vector и LightHouse.

## Репозитории

| Назначение | Репозиторий |
|---|---|
| Vector role | <https://github.com/victoryurochkin/vector-role> |
| LightHouse role | <https://github.com/victoryurochkin/lighthouse-role> |
| Основной playbook | <https://github.com/victoryurochkin/08-ansible-04-role> |

## Используемая инфраструктура

Для выполнения задания использовались две виртуальные машины с Ubuntu 22.04.

| Hostname | IP-адрес | Назначение |
|---|---:|---|
| `ansible` | `192.168.1.90` | Управляющий сервер Ansible |
| `target-server` | `192.168.1.84` | Целевой сервер для установки ClickHouse, Vector и LightHouse |

<img width="1897" height="1420" alt="image" src="https://github.com/user-attachments/assets/39bb6457-ebc5-49c4-aec9-565fb871976d" />

На управляющем сервере установлен Ansible и настроен SSH-доступ к целевому серверу.

## Проверка Ansible

На управляющем сервере была выполнена проверка доступности целевого хоста:

```bash
ansible -i inventory/prod.yml all -m ping
```

Результат:

```text
target-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Также была проверена возможность выполнения команд с повышением привилегий:

```bash
ansible -i inventory/prod.yml all -m command -a "whoami" -b
```

Результат:

```text
target-server | CHANGED | rc=0 >>
root
```

<img width="1786" height="893" alt="image" src="https://github.com/user-attachments/assets/b2ba65fd-6924-4adb-a1a8-4f65385fbafa" />


## Структура основного репозитория

Основной репозиторий содержит playbook, inventory, переменные и файл зависимостей roles:

```text
08-ansible-04-role/
├── group_vars
│   └── all.yml
├── inventory
│   └── prod.yml
├── README.md
├── requirements.yml
└── site.yml
```

<img width="1133" height="1187" alt="image" src="https://github.com/user-attachments/assets/759a8649-cfec-4557-8ac7-8ed1f3092545" />


Каталог `roles/` не хранится в основном репозитории и добавлен в `.gitignore`, так как роли устанавливаются через `ansible-galaxy` из `requirements.yml`.

## Файл inventory

Файл `inventory/prod.yml`:

```yaml
---
all:
  hosts:
    target-server:
      ansible_host: 192.168.1.84
      ansible_user: it
      ansible_become: true
```

## Файл переменных

Файл `group_vars/all.yml`:

```yaml
---
clickhouse_version: "22.3.3.44"
clickhouse_packages:
  - clickhouse-client
  - clickhouse-server
  - clickhouse-common-static

vector_clickhouse_endpoint: "http://localhost:8123"
vector_clickhouse_database: "logs"
vector_clickhouse_table: "vector_logs"

lighthouse_listen_port: 80
lighthouse_server_name: "_"
```

## Файл requirements.yml

В файл `requirements.yml` добавлены роли ClickHouse, Vector и LightHouse.

```yaml
---
- src: https://github.com/AlexeySetevoi/ansible-clickhouse.git
  scm: git
  version: "1.13"
  name: clickhouse

- src: https://github.com/victoryurochkin/vector-role.git
  scm: git
  version: "1.0.0"
  name: vector-role

- src: https://github.com/victoryurochkin/lighthouse-role.git
  scm: git
  version: "1.0.0"
  name: lighthouse-role
```

Роли Vector и LightHouse опубликованы в отдельных публичных репозиториях и имеют теги `1.0.0`.

## Установка roles

Перед запуском playbook роли устанавливаются командой:

```bash
ansible-galaxy install -r requirements.yml -p roles
```

Результат установки:

```text
Starting galaxy role install process
- extracting clickhouse to /root/ansible-hw-04/roles/clickhouse
- clickhouse (1.13) was installed successfully
- extracting vector-role to /root/ansible-hw-04/roles/vector-role
- vector-role (1.0.0) was installed successfully
- extracting lighthouse-role to /root/ansible-hw-04/roles/lighthouse-role
- lighthouse-role (1.0.0) was installed successfully
```

После установки роли располагаются в каталоге `roles/`:

```text
roles
├── clickhouse
├── lighthouse-role
└── vector-role
```

## Основной playbook

Файл `site.yml`:

```yaml
---
- name: Install ClickHouse, Vector and LightHouse
  hosts: all
  become: true

  roles:
    - role: clickhouse
    - role: vector-role
    - role: lighthouse-role
```

## Роль vector-role

Роль `vector-role` выполняет:

- установку зависимостей;
- скачивание deb-пакета Vector;
- установку Vector;
- создание каталога конфигурации;
- генерацию файла `/etc/vector/vector.yaml` из шаблона;
- запуск и включение сервиса Vector.

Основные переменные роли:

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `vector_version` | `0.38.0` | Версия Vector |
| `vector_arch` | `amd64` | Архитектура пакета |
| `vector_config_dir` | `/etc/vector` | Каталог конфигурации |
| `vector_config_file` | `/etc/vector/vector.yaml` | Основной конфигурационный файл |
| `vector_service_name` | `vector` | Имя systemd-сервиса |
| `vector_clickhouse_endpoint` | `http://localhost:8123` | HTTP endpoint ClickHouse |
| `vector_clickhouse_database` | `logs` | База данных ClickHouse |
| `vector_clickhouse_table` | `vector_logs` | Таблица ClickHouse |

## Роль lighthouse-role

Роль `lighthouse-role` выполняет:

- установку зависимостей;
- установку nginx;
- клонирование репозитория LightHouse;
- генерацию nginx-конфигурации из шаблона;
- включение сайта LightHouse;
- отключение стандартного сайта nginx;
- запуск и включение сервиса nginx.

Основные переменные роли:

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `lighthouse_repo` | `https://github.com/VKCOM/lighthouse.git` | Репозиторий LightHouse |
| `lighthouse_version` | `master` | Ветка или тег LightHouse |
| `lighthouse_dest` | `/var/www/lighthouse` | Каталог установки LightHouse |
| `lighthouse_nginx_conf` | `/etc/nginx/sites-available/lighthouse.conf` | Путь к nginx-конфигурации |
| `lighthouse_nginx_enabled_conf` | `/etc/nginx/sites-enabled/lighthouse.conf` | Путь к активной nginx-конфигурации |
| `lighthouse_listen_port` | `80` | Порт nginx |
| `lighthouse_server_name` | `_` | Имя сервера nginx |

## Проверка синтаксиса

Перед запуском playbook была выполнена проверка синтаксиса:

```bash
ansible-playbook -i inventory/prod.yml site.yml --syntax-check
```

Результат:

```text
playbook: site.yml
```

## Запуск playbook

Playbook запускается командой:

```bash
ansible-playbook -i inventory/prod.yml site.yml
```

Результат выполнения:

```text
PLAY RECAP ************************************************************************************************************************
target-server              : ok=41 changed=21 unreachable=0 failed=0 skipped=10 rescued=0 ignored=0
```

После первичной установки был выполнен повторный запуск playbook для проверки идемпотентности.

Результат повторного запуска:

```text
PLAY RECAP ************************************************************************************************************************
target-server              : ok=38 changed=0 unreachable=0 failed=0 skipped=10 rescued=0 ignored=0
```

Повторный запуск завершился без изменений, что подтверждает корректную идемпотентность playbook.

## Проверка сервисов

После выполнения playbook были проверены сервисы ClickHouse, Vector и nginx.

```bash
ansible -i inventory/prod.yml all -m command -a "systemctl is-active clickhouse-server" -b
ansible -i inventory/prod.yml all -m command -a "systemctl is-active vector" -b
ansible -i inventory/prod.yml all -m command -a "systemctl is-active nginx" -b
```

Результат:

```text
target-server | CHANGED | rc=0 >>
active

target-server | CHANGED | rc=0 >>
active

target-server | CHANGED | rc=0 >>
active
```

## Проверка портов

Была выполнена проверка открытых портов:

```bash
ansible -i inventory/prod.yml all -m shell -a "ss -tulpn | grep -E '8123|9000|80'" -b
```

Результат:

```text
tcp   LISTEN 0      4096            127.0.0.1:9000      0.0.0.0:*    users:(("clickhouse-serv",pid=19057,fd=160))
tcp   LISTEN 0      511               0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=21263,fd=6),("nginx",pid=21262,fd=6),("nginx",pid=21261,fd=6),("nginx",pid=21260,fd=6),("nginx",pid=20955,fd=6))
tcp   LISTEN 0      4096            127.0.0.1:8123      0.0.0.0:*    users:(("clickhouse-serv",pid=19057,fd=159))
tcp   LISTEN 0      4096                [::1]:8123         [::]:*    users:(("clickhouse-serv",pid=19057,fd=71))
tcp   LISTEN 0      4096                [::1]:9000         [::]:*    users:(("clickhouse-serv",pid=19057,fd=73))
```

## Проверка ClickHouse

Проверка доступности ClickHouse:

```bash
ansible -i inventory/prod.yml all -m command -a "clickhouse-client -q 'SHOW DATABASES'" -b
```

Результат:

```text
target-server | CHANGED | rc=0 >>
INFORMATION_SCHEMA
default
information_schema
system
```

## Проверка LightHouse

Проверка доступности LightHouse через nginx:

```bash
curl -I http://192.168.1.84/
```

Результат:

```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 27 May 2026 08:03:46 GMT
Content-Type: text/html
Content-Length: 6766
Last-Modified: Wed, 27 May 2026 08:02:41 GMT
Connection: keep-alive
ETag: "6a16a521-1a6e"
Accept-Ranges: bytes
```

<img width="2307" height="732" alt="image" src="https://github.com/user-attachments/assets/10b0584b-6f94-48c5-9013-b956f7b9aa0a" />

<img width="3071" height="798" alt="image" src="https://github.com/user-attachments/assets/54a3ff1e-5e85-4630-93d5-3119ef5a11b5" />

## Итог

В результате выполнения домашнего задания:

- создана отдельная роль `vector-role`;
- создана отдельная роль `lighthouse-role`;
- использована готовая роль `clickhouse`;
- роли Vector и LightHouse вынесены в отдельные публичные репозитории;
- для ролей Vector и LightHouse проставлены теги `1.0.0`;
- основной playbook переработан на использование roles;
- зависимости roles описаны в `requirements.yml`;
- playbook успешно выполняется на целевом сервере;
- повторный запуск playbook проходит без изменений;
- ClickHouse, Vector и LightHouse успешно установлены и запущены.

## Ссылки для проверки

- Vector role: <https://github.com/victoryurochkin/vector-role>
- LightHouse role: <https://github.com/victoryurochkin/lighthouse-role>
- Основной playbook: <https://github.com/victoryurochkin/08-ansible-04-role>
