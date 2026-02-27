# Лабораторная работа по BeeGFS

https://doc.beegfs.io/

Основные теоретические сведения

1. Назначение BeeGFS в HPC

BeeGFS - параллельная распределённая файловая система для организации общего
файлового пространства (shared storage), доступного одновременно с нескольких узлов.
Типовые сценарии:
• единое хранилище для login-node и вычислительных узлов;
• хранение входных данных, результатов расчётов, checkpoint’ов;
• параллельный доступ к данным при интенсивном I/O.


2. Архитектура BeeGFS

BeeGFS состоит из сервисов, которые могут быть размещены как на отдельных узлах, так
и совмещаться:
• Management service - реестр компонентов BeeGFS и координация;
• Metadata service - для операций с метаданными (каталоги, имена файлов, права
доступа);
• Storage service - сервис данных файлов;
• Client - клиент.


3. Особенности производительности

Производительность BeeGFS зависит от:
• пропускной способности и задержек сети;
• количества storage targets и их производительности;
• нагрузки на metadata (критично при большом количестве мелких файлов).


4. Замечание о надёжности хранения

BeeGFS не является RAID-системой. При отказе диска данные на соответствующем storage
target могут быть потеряны, если не используются дополнительные механизмы:
• аппаратный RAID/СХД;
• mirroring/replication BeeGFS (production-конфигурации);
• резервное копирование.



Развёртывание BeeGFS
Состав стенда
| Имя ВМ | Роль | Точка монтирования |
|--------|------|--------------------|
| nodeA  | • management service<br>• metadata service<br>• storage service<br>• клиент BeeGFS | /mnt/beegfs |
| nodeB  | • storage service<br>• клиент BeeGFS | /mnt/beegfs |

| Этап | Задание | Действия | Результат / Примечание |
|------|----------|----------|------------------------|
| Подготовка ОС | Установить базовые утилиты | sudo apt update<br>sudo apt -y install chrony vim net-tools jq<br>sudo systemctl enable --now chrony | Утилиты установлены |
| Подготовка ОС | Установить зависимости | sudo apt -y install linux-headers-$(uname -r) gcc make dkms | Зависимости установлены |
| Установка BeeGFS | Установка пакетов (nodeA) | sudo apt -y install \<br>beegfs-mgmtd beegfs-meta beegfs-storage \<br>beegfs-client beegfs-utils | Пакеты установлены (предварительно требуется настроить репозиторий BeeGFS) |
| Установка BeeGFS | Установка пакетов (nodeB) | sudo apt -y install \<br>beegfs-storage \<br>beegfs-client beegfs-utils | Пакеты установлены |
| Подготовка дисков | Подключить дополнительные диски | • nodeA — диск для storage target<br>• nodeB — диск для storage target | Диски подключены |
| Подготовка дисков | Проверка путей by-id | ls -l /dev/disk/by-id/<br><br>lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL,SERIAL<br>ls -l /dev/disk/by-id/ | Диски доступны по by-id |
| Storage targets | Форматирование и монтирование (nodeA) | sudo mkfs.xfs /dev/disk/by-id/<DISK_A_ID><br>sudo mkdir -p /data/beegfs_storage<br>sudo mount /dev/disk/by-id/<DISK_A_ID> /data/beegfs_storage<br>df -hT \| grep /data/beegfs_storage | Раздел смонтирован |
| Storage targets | Форматирование и монтирование (nodeB) | sudo mkfs.xfs /dev/disk/by-id/<DISK_B_ID><br>sudo mkdir -p /data/beegfs_storage<br>sudo mount /dev/disk/by-id/<DISK_B_ID> /data/beegfs_storage<br>df -hT \| grep /data/beegfs_storage | Раздел смонтирован |
| Storage targets | Добавить в fstab (при необходимости) | Использовать путь /dev/disk/by-id/<DISK_ID> в /etc/fstab | Автоматическое монтирование после перезагрузки |


## Первичная настройка сервисов BeeGFS

| Задание | Порядок действия | Ожидаемый результат | Примечание |
|----------|------------------|---------------------|------------|
| Management service (nodeA) | На nodeA:<br><br>sudo mkdir -p /data/beegfs_mgmtd<br>sudo /opt/beegfs/sbin/beegfs-setup-mgmtd -p /data/beegfs_mgmtd | Инициализирован management service, создан конфигурационный каталог | Выполняется только на nodeA |
| Metadata service (nodeA) | На nodeA:<br><br>sudo mkdir -p /data/beegfs_meta<br>sudo /opt/beegfs/sbin/beegfs-setup-meta -p /data/beegfs_meta -s 1 -m nodeA | Metadata target зарегистрирован в management service | ID metadata: 1 |
| Storage service (nodeA) | На nodeA:<br><br>sudo /opt/beegfs/sbin/beegfs-setup-storage -p /data/beegfs_storage -s 1 -i 101 -m nodeA | Storage target зарегистрирован в management service | Storage ID: 101, storage node ID: 1 |
| Storage service (nodeB) | На nodeB:<br><br>sudo /opt/beegfs/sbin/beegfs-setup-storage -p /data/beegfs_storage -s 2 -i 201 -m nodeA | Storage target зарегистрирован в management service | Storage ID: 201, storage node ID: 2, management находится на nodeA |

## Настройка аутентификации соединений. Вариант A: connAuthFile

| Задание | Порядок действия | Ожидаемый результат | Примечание |
|----------|------------------|---------------------|------------|
| Создать connAuthFile (nodeA) | На nodeA:<br><br>sudo install -d -m 0755 /etc/beegfs<br>sudo bash -c 'umask 077; head -c 64 /dev/urandom \| base64 > /etc/beegfs/connauthfile'<br>sudo chmod 600 /etc/beegfs/connauthfile | Создан файл аутентификации с корректными правами (600) | Файл должен быть защищён от чтения другими пользователями |
| Скопировать connauthfile на nodeB | На nodeA:<br><br>scp /etc/beegfs/connauthfile <user>@<nodeB>:/etc/beegfs/connauthfile | Файл аутентификации присутствует на nodeB | На nodeB также должны быть права 600 |
| Включить использование connAuthFile | На nodeA:<br><br>sudo sed -i 's|^#\?connAuthFile.*|connAuthFile=/etc/beegfs/connauthfile|' /etc/beegfs/beegfs-* | В конфигурационных файлах BeeGFS прописан параметр connAuthFile | Аналогично параметр должен быть установлен на nodeB (если не используется централизованная конфигурация) |


## Настройка аутентификации соединений. Вариант B: отключить аутентификацию
| Задание | Порядок действий |
|----------|------------------|
| Отключить аутентификацию соединений | На nodeA и nodeB:<br><br>
```
sudo sed -i 's|^#\?connDisableAuthentication.*|connDisableAuthentication=true|' /etc/beegfs/beegfs-* |
```


| Задание | Порядок действия | Ожидаемый результат | Примечание |
|----------|------------------|---------------------|------------|
| Запустить сервисы (nodeA) | На nodeA:<br><br>sudo systemctl enable --now beegfs-mgmtd<br>sudo systemctl enable --now beegfs-meta<br>sudo systemctl enable --now beegfs-storage<br>sudo systemctl enable --now beegfs-client | Все сервисы запущены и активны (active/running) | Проверить статус: systemctl status beegfs-* |
| Запустить сервисы (nodeB) | На nodeB:<br><br>sudo systemctl enable --now beegfs-storage<br>sudo systemctl enable --now beegfs-client | Сервисы запущены и активны (active/running) | В исходной команде была опечатка: beegfs-cliente → beegfs-client |
| Проверить монтирование клиента | На обоих узлах:<br><br>mount \| grep beegfs<br>df -hT \| grep beegfs | Файловая система BeeGFS смонтирована и отображается в списке | Обычно монтируется в /mnt/beegfs (если использовался путь по умолчанию) |

| Задание | Порядок действия | Ожидаемый результат | Примечание |
|----------|------------------|---------------------|------------|
| Проверка сервисов | sudo systemctl status beegfs-mgmtd<br>sudo systemctl status beegfs-meta<br>sudo systemctl status beegfs-storage<br>sudo systemctl status beegfs-client | Все сервисы в состоянии active (running), ошибок нет | Проверять на соответствующих узлах |
| Проверка состояния компонентов (nodeB) | На nodeB:<br><br>beegfs health check<br>beegfs health capacity<br>beegfs node list --with-nics | Все узлы отображаются, состояние healthy, доступна информация по capacity | Позволяет проверить сетевые интерфейсы и статус targets |
| Перезагрузка storage-узла (nodeB) | 1. sudo reboot<br>2. После загрузки:<br>sudo systemctl status beegfs-storage<br>mount \| grep beegfs | Storage сервис запущен, клиентская ФС смонтирована | Проверить автозапуск сервисов |
| Перезагрузка management/metadata-узла (nodeA) | 1. sudo reboot<br>2. После загрузки:<br>sudo systemctl status beegfs-mgmtd<br>sudo systemctl status beegfs-meta<br>sudo systemctl status beegfs-storage<br>sudo systemctl status beegfs-client | Все сервисы запущены без ошибок | Проверить корректность регистрации узлов после старта |
| Замена диска storage target (nodeB) | 1. sudo systemctl stop beegfs-storage<br>2. sudo umount /data/beegfs_storage<br>3. lsblk; ls -l /dev/disk/by-id/<br>4. sudo mkfs.xfs /dev/disk/by-id/<NEW_DISK_B_ID><br>   sudo mount /dev/disk/by-id/<NEW_DISK_B_ID> /data/beegfs_storage<br>   df -hT \| grep /data/beegfs_storage<br>5. sudo systemctl start beegfs-storage<br>   sudo systemctl status beegfs-storage<br>6. Проверка с nodeA:<br>   beegfs health check<br>   beegfs health capacity | Новый диск смонтирован, storage target активен и отображается в health check | Важно использовать путь by-id и корректный target ID |
| Диагностика при отсутствии монтирования BeeGFS | 1. На клиенте:<br>sudo systemctl status beegfs-client<br>mount \| grep beegfs<br>2. На серверах:<br>sudo systemctl status beegfs-mgmtd<br>sudo systemctl status beegfs-meta<br>sudo systemctl status beegfs-storage<br>3. Сетевые проверки:<br>beegfs health check<br>beegfs node list --with-nics | Выявлена причина отсутствия монтирования (сервис, сеть или management) | Проверять по цепочке: клиент → management → meta → storage → сеть |

