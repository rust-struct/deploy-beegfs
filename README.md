# BeeGFS Ansible — документация

## Состав стенда

| Узел  | Сервисы BeeGFS                              | Точка монтирования |
|-------|---------------------------------------------|--------------------|
| nodeA | management + metadata + storage + client    | /mnt/beegfs        |
| nodeB | storage + client                            | /mnt/beegfs        |

---

## Предварительные требования

### 1. Виртуальные машины

| Параметр        | Минимум          | Примечание                              |
|-----------------|------------------|-----------------------------------------|
| ОС              | Ubuntu 22.04 LTS | Для Ubuntu 24.04 (noble) см. примечание о репозитории |
| CPU             | 2 vCPU           |                                         |
| RAM             | 2 ГБ             |                                         |
| Системный диск  | 20 ГБ            |                                         |
| Доп. диск       | по 1 на каждом узле | Для storage target (любой размер)    |

> **Ubuntu 24.04 (noble):** репозиторий BeeGFS может не иметь сборки под `noble`.
> В этом случае задайте `beegfs_dist_codename: "jammy"` в `inventory/host_vars/nodeA/main.yml`
> и `inventory/host_vars/nodeB/main.yml`.

### 2. Диски

На каждой ВМ должен быть подключён **дополнительный диск** для storage target.
Диск должен быть:
- подключён к ВМ (виден в `lsblk`)
- **не отформатирован** и **не смонтирован** (роль сделает это сама, если задана переменная `beegfs_storage_device`)
- либо уже отформатирован в XFS и смонтирован в `/data/beegfs_storage` вручную — тогда `beegfs_storage_device` оставьте пустым

Узнать путь диска по-id перед заполнением переменных:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,SERIAL
ls -l /dev/disk/by-id/
```

Пример вывода, по которому нужно выбрать путь:
```
/dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi1 -> ../../sdb
```

### 3. Сеть

- Все узлы должны **пинговаться между собой** по имени хоста (`nodeA`, `nodeB`) или по IP.
- Имена хостов должны разрешаться — либо через DNS, либо через `/etc/hosts`.
- Порты BeeGFS (TCP/UDP **8008, 8003, 8004**) должны быть открыты между всеми узлами.
- Ansible-контроллер должен иметь SSH-доступ к обоим узлам.

Быстрая проверка доступности хостов:
```bash
ansible -i inventory/hosts.ini beegfs -m ping
```

### 4. SSH и sudo

На обоих узлах должен существовать пользователь с:
- доступом по SSH (по ключу — рекомендуется)
- **беспарольным sudo** (либо пароль передаётся через `-K`)

Пример настройки беспарольного sudo:
```bash
echo "ubuntu ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ubuntu
```

### 5. Ansible на контроллере

```bash
pip install ansible
# или
sudo apt install ansible
```

Проверка версии (требуется >= 2.12):
```bash
ansible --version
```

---

## Настройка перед запуском

### Шаг 1 — Заполнить IP-адреса в инвентаре

Файл: `inventory/hosts.ini`

```ini
[beegfs_mgmtd]
nodeA ansible_host=<IP_nodeA>   # ← вставить реальный IP

[beegfs_storage]
nodeA
nodeB ansible_host=<IP_nodeB>   # ← вставить реальный IP
```

### Шаг 2 — Задать пути к дискам в host_vars

Если роль должна сама форматировать и монтировать диски, заполните
`beegfs_storage_device` в host_vars каждого узла.

Файл: `inventory/host_vars/nodeA/main.yml`
```yaml
beegfs_storage_device: "/dev/disk/by-id/<DISK_A_ID>"
```

Файл: `inventory/host_vars/nodeB/main.yml`
```yaml
beegfs_storage_device: "/dev/disk/by-id/<DISK_B_ID>"
```

Если диски уже смонтированы вручную — оставьте `beegfs_storage_device: ""`
и убедитесь, что они смонтированы в `/data/beegfs_storage`.

### Шаг 3 — Выбрать режим аутентификации

В `roles/beegfs/defaults/main.yml`:

| Переменная           | Значение | Поведение                                         |
|----------------------|----------|---------------------------------------------------|
| `beegfs_disable_auth`| `true`   | Аутентификация отключена (вариант B, для лабы)    |
| `beegfs_disable_auth`| `false`  | Генерируется `connAuthFile` и раздаётся всем узлам|

Для лабораторной среды рекомендуется `true`.

---

## Запуск

### Деплой

```bash
ansible-playbook -i inventory/hosts.ini playbook.yml
```

С запросом sudo-пароля (если беспарольный sudo не настроен):
```bash
ansible-playbook -i inventory/hosts.ini playbook.yml -K
```

### Верификация

```bash
ansible-playbook -i inventory/hosts.ini verify.yml
```

---

## Структура проекта

```
.
├── playbook.yml              # Деплой
├── verify.yml                # Верификация
├── inventory/
│   ├── hosts.ini             # Инвентарь
│   └── host_vars/
│       ├── nodeA/main.yml    # Переменные nodeA (disk, node IDs)
│       └── nodeB/main.yml    # Переменные nodeB (disk, node IDs)
└── roles/beegfs/
    ├── defaults/main.yml     # Переменные по умолчанию
    ├── handlers/main.yml     # Рестарт сервисов
    ├── meta/main.yml
    └── tasks/
        ├── main.yml          # Точка входа
        ├── repo.yml          # Добавление репозитория BeeGFS
        ├── install.yml       # Установка пакетов и зависимостей ОС
        ├── storage_mount.yml # Форматирование и монтирование диска (опц.)
        ├── setup_mgmtd.yml   # Инициализация management service
        ├── setup_meta.yml    # Инициализация metadata service
        ├── setup_storage.yml # Инициализация storage service
        ├── auth.yml          # Настройка аутентификации
        ├── client_config.yml # Настройка клиента (sysMgmtdHost, mounts)
        └── services.yml      # Запуск и autostart сервисов
```

---

## Типичные проблемы

### BeeGFS не монтируется на клиенте

Проверить по цепочке согласно лабе:

```bash
# 1. Клиент
sudo systemctl status beegfs-client
mount | grep beegfs

# 2. Серверы
sudo systemctl status beegfs-mgmtd   # только nodeA
sudo systemctl status beegfs-meta    # только nodeA
sudo systemctl status beegfs-storage # оба узла

# 3. Сеть / кластер
beegfs health check
beegfs node list --with-nics
```

### Репозиторий BeeGFS недоступен для Ubuntu 24.04

Задайте явно codename Jammy:
```yaml
# inventory/host_vars/nodeA/main.yml и nodeB/main.yml
beegfs_dist_codename: "jammy"
```

### beegfs-setup-* завершается с ошибкой «already initialized»

Директория уже содержит `format.conf` — это нормально, команда пропускается
(задача идемпотентна через `creates: format.conf`).
