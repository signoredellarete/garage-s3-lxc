# Garage S3 — Installation and Configuration Runbook

**Environment**: LXC Ubuntu 24.04  
**Garage version**: v2.2.0  
**Official documentation**: https://garagehq.deuxfleurs.fr/documentation/

---

## Prerequisites

- LXC Ubuntu 24.04 with two network interfaces
- Root access to the container
- `garage` binary already downloaded

---

## 1. Install the binary

```bash
chmod +x garage
mv garage /usr/local/bin/garage
garage --version
```

---

## 2. Configuration

Create `/etc/garage.toml`:

```toml
metadata_dir = "/var/lib/garage/meta"
data_dir = "/var/lib/garage/data"
db_engine = "lmdb"

replication_factor = 1

rpc_bind_addr = "[::]:3901"
rpc_public_addr = "10.1.0.210:3901"
rpc_secret = "<output of: openssl rand -hex 32>"

[s3_api]
s3_region = "garage"
api_bind_addr = "10.1.0.210:3900"
root_domain = ".s3.garage.local"

[s3_web]
bind_addr = "[::]:3902"
root_domain = ".web.garage.local"
index = "index.html"

[admin]
api_bind_addr = "127.0.0.1:3903"
admin_token = "<output of: openssl rand -base64 32>"
metrics_token = "<output of: openssl rand -base64 32>"
```

Generate the secret values:

```bash
openssl rand -hex 32      # for rpc_secret
openssl rand -base64 32   # for admin_token
openssl rand -base64 32   # for metrics_token
```

> **Note**: save `admin_token` — it is used by the WebUI to authenticate against the administration API.

---

## 3. Garage systemd service

Create `/etc/systemd/system/garage.service`:

```ini
[Unit]
Description=Garage Data Store
After=network-online.target
Wants=network-online.target

[Service]
Environment='RUST_LOG=garage=info' 'RUST_BACKTRACE=1'
ExecStart=/usr/local/bin/garage server
StateDirectory=garage
DynamicUser=true
ProtectHome=true
NoNewPrivileges=true
LimitNOFILE=42000

[Install]
WantedBy=multi-user.target
```

> **Important**: `DynamicUser=true` automatically creates the service user and maps `/var/lib/garage` to `/var/lib/private/garage`. Do not manually create the `/var/lib/garage` directory before the first start.

Enable and start:

```bash
systemctl daemon-reload
systemctl enable --now garage
systemctl status garage
```

---

## 4. Cluster initialization

```bash
# Check that the node is up
garage status
```

The output will show the node with status `NO ROLE ASSIGNED`. Copy the node ID (hex string) and assign the layout:

```bash
garage layout assign -z dc1 -c <CAPACITY> <NODE_ID>
# example: garage layout assign -z dc1 -c 500G abc123def456...

garage layout apply --version 1
garage status
```

The node should now appear with the assigned capacity and zone `dc1`.

---

## 5. WebUI installation

Download the binary from https://github.com/khairul169/garage-webui/releases:

```bash
wget -O garage-webui \
  https://github.com/khairul169/garage-webui/releases/download/1.1.0/garage-webui-v1.1.0-linux-amd64
chmod +x garage-webui
mv garage-webui /usr/local/bin/garage-webui
```

Create `/etc/systemd/system/garage-webui.service`:

```ini
[Unit]
Description=Garage Web UI
After=garage.service

[Service]
Environment="PORT=80"
Environment="CONFIG_PATH=/etc/garage.toml"
ExecStart=/usr/local/bin/garage-webui
AmbientCapabilities=CAP_NET_BIND_SERVICE
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable --now garage-webui
```

The WebUI is reachable at `http://10.1.0.210` and `http://192.168.1.41`.

---

## 6. Bucket, access key and permissions

### 6.1 Create the bucket

To ensure compatibility with S3 clients using path-style addressing and no versioning support:

```bash
garage bucket create <BUCKET_NAME>
```

Verify:

```bash
garage bucket list
```

### 6.2 Create the access key

```bash
garage key create <KEY_NAME>
```

The output returns the `Key ID` and `Secret key`. **Save them immediately** — the secret cannot be retrieved afterwards.

```bash
# To display the Key ID again (secret not shown)
garage key info <KEY_NAME>
```

### 6.3 Grant access to the bucket

```bash
garage bucket allow \
  --read \
  --write \
  --owner \
  <BUCKET_NAME> \
  --key <KEY_NAME>
```

### 6.4 Verify

```bash
garage bucket info <BUCKET_NAME>
```

The output should show the associated key with `R/W/Owner` permissions.

---

## Port reference

| Port | Interface | Service |
|------|-----------|---------|
| 3900 | `10.1.0.210` | S3 API |
| 3901 | `[::]` | Inter-node RPC (internal) |
| 3902 | `[::]` | S3 Web hosting |
| 3903 | `127.0.0.1` | Admin API (local only) |
| 80   | `[::]` | WebUI |

---

## References

- [Quick Start](https://garagehq.deuxfleurs.fr/documentation/quick-start/)
- [Systemd](https://garagehq.deuxfleurs.fr/documentation/cookbook/systemd/)
- [Configuration reference](https://garagehq.deuxfleurs.fr/documentation/reference-manual/configuration/)
- [garage-webui](https://github.com/khairul169/garage-webui)
