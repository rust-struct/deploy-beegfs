# BeeGFS TLS: Manual Enablement (CLI on Nodes)

This guide enables TLS manually after playbooks are already applied.
No Ansible commands are used below.

## Nodes

- `nodeA` (management/meta/storage/client): `172.24.14.40`
- `nodeB` (storage/client): `172.24.14.41`

## 1) Generate certificate and key on nodeA

Login to `nodeA` and run:

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

## 2) Enable TLS in mgmtd config on nodeA

On `nodeA` edit `/etc/beegfs/beegfs-mgmtd.toml`:

```bash
sudo vi /etc/beegfs/beegfs-mgmtd.toml
```

Set/add these lines:

```toml
tls-disable = false
tls-cert-file = "/etc/beegfs/cert.pem"
tls-key-file = "/etc/beegfs/key.pem"
```

Restart mgmtd:

```bash
sudo systemctl restart beegfs-mgmtd
sudo systemctl is-active beegfs-mgmtd
```

Expected output: `active`.

## 3) Copy certificate to nodeB (for CLI verification)

Only `cert.pem` is needed on `nodeB` (never copy `key.pem`).

From `nodeA`:

```bash
scp /etc/beegfs/cert.pem root@172.24.14.41:/etc/beegfs/cert.pem
```

Then on `nodeB`:

```bash
sudo chown root:root /etc/beegfs/cert.pem
sudo chmod 644 /etc/beegfs/cert.pem
```

## 4) Validate TLS mode from nodeA

Run on `nodeA` (without `--tls-disable`):

```bash
sudo beegfs health check
sudo beegfs health capacity
sudo beegfs node list --with-nics
```

If cert name mismatch occurs, temporary check:

```bash
sudo beegfs --tls-disable-verification health check
```

Then regenerate certificate with correct SAN and remove `--tls-disable-verification`.

## 5) Optional validation from nodeB

```bash
sudo beegfs --mgmtd-addr 172.24.14.40:8010 health check
```

## Rollback to non-TLS lab mode

On `nodeA`:

```bash
sudo sed -i 's/^tls-disable\s*=.*/tls-disable = true/' /etc/beegfs/beegfs-mgmtd.toml
sudo systemctl restart beegfs-mgmtd
```

Then use:

```bash
sudo beegfs --tls-disable health check
```
