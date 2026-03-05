# BeeGFS TLS: ручное включение (CLI на самих нодах)

Инструкция включает TLS вручную после того, как плейбуки уже отработали.
Ниже только команды, выполняемые на самих узлах, без Ansible.

## Узлы

- `nodeA` (management/meta/storage/client): `172.24.14.40`
- `nodeB` (storage/client): `172.24.14.41`

## 1) Сгенерировать сертификат и ключ на nodeA

Зайди на `nodeA` и выполни:

```bash
sudo install -d -m 0755 /etc/beegfs

sudo openssl req -x509 -nodes -newkey rsa:4096 -days 825 \
  -keyout /etc/beegfs/key.pem \
  -out /etc/beegfs/cert.pem \
  -subj "/CN=172.24.14.40" \
  -addext "subjectAltName=IP:172.24.14.40,DNS:nodeA"

sudo chmod 600 /etc/beegfs/key.pem
sudo chmod 644 /etc/beegfs/cert.pem
```

## 2) Включить TLS в конфиге mgmtd на nodeA

Открой `/etc/beegfs/beegfs-mgmtd.toml`:

```bash
sudo vi /etc/beegfs/beegfs-mgmtd.toml
```

Проверь/добавь строки:

```toml
tls-disable = false
tls-cert-file = "/etc/beegfs/cert.pem"
tls-key-file = "/etc/beegfs/key.pem"
```

Перезапусти сервис:

```bash
sudo systemctl restart beegfs-mgmtd
sudo systemctl is-active beegfs-mgmtd
```

Ожидаемый результат: `active`.

## 3) Скопировать сертификат на nodeB (для проверки CLI)

На `nodeB` нужен только `cert.pem`. Приватный ключ `key.pem` копировать нельзя.

С `nodeA`:

```bash
scp /etc/beegfs/cert.pem root@172.24.14.41:/etc/beegfs/cert.pem
```

На `nodeB`:

```bash
sudo chown root:root /etc/beegfs/cert.pem
sudo chmod 644 /etc/beegfs/cert.pem
```

## 4) Проверить работу без `--tls-disable`

На `nodeA`:

```bash
sudo beegfs health check
sudo beegfs health capacity
sudo beegfs node list --with-nics
```

Если будет ошибка несоответствия имени/адреса в сертификате, временно можно проверить так:

```bash
sudo beegfs --tls-disable-verification health check
```

После этого лучше перевыпустить сертификат с корректным `SAN`, чтобы убрать `--tls-disable-verification`.

## 5) Дополнительная проверка с nodeB

```bash
sudo beegfs --mgmtd-addr 172.24.14.40:8010 health check
```

## Откат обратно в lab-режим без TLS

На `nodeA`:

```bash
sudo sed -i 's/^tls-disable\s*=.*/tls-disable = true/' /etc/beegfs/beegfs-mgmtd.toml
sudo systemctl restart beegfs-mgmtd
```

После отката команды снова запускать с `--tls-disable`, например:

```bash
sudo beegfs --tls-disable health check
```
