# Ansible: Multi-HTTPD + HAProxy

Deploys **N independent Apache HTTPD instances** (overcoming the single-process thread limit) fronted by **HAProxy** as an HTTPS load balancer.

## Problem This Solves

Apache HTTPD has a hard ceiling on the number of threads it can create (controlled by `MaxRequestWorkers`). Once that limit is hit, new incoming connections are queued or dropped — existing users get slow responses and new users cannot connect at all.

**Solution:** Instead of one `httpd` process, run many — each with its own config, PID file, log directory, and `systemd` service (`httpd1`, `httpd2`, …, `httpdN`). HAProxy round-robins HTTPS traffic across all of them, multiplying your effective capacity by N.

```
Internet (HTTPS :443)
        │
   ┌────▼────┐
   │ HAProxy │   ← TLS termination, round-robin
   └────┬────┘
   ┌────┴──────────────────────┐
   │                           │
┌──▼──┐  ┌──▼──┐  … ┌──────┐
│httpd1│  │httpd2│     │httpdN│
│:8080 │  │:8081 │     │:808N │
└──────┘  └──────┘     └──────┘
```

---

## Quick Start

```bash
# 1. Copy the example inventory and customise it
cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml
$EDITOR ansible/inventory/hosts.yml

# 2. Run the playbook (default: 3 instances)
cd ansible
ansible-playbook -i inventory/hosts.yml site.yml

# 3. Override the instance count on the fly
ansible-playbook -i inventory/hosts.yml site.yml -e "httpd_instance_count=10"
```

---

## Directory Layout

```
ansible/
├── ansible.cfg                     # Defaults (inventory path, sudo, output format)
├── site.yml                        # Top-level playbook
├── inventory/
│   └── hosts.yml.example           # Copy to hosts.yml and edit
├── group_vars/
│   └── all.yml                     # All variable defaults (see table below)
├── roles/
│   ├── multi_httpd/
│   │   ├── defaults/main.yml       # Role defaults
│   │   ├── tasks/
│   │   │   ├── main.yml            # Orchestration (create dirs, templates, services)
│   │   │   ├── install_redhat.yml  # dnf/yum install for RHEL/Rocky/Alma
│   │   │   └── install_debian.yml  # apt install for Ubuntu/Debian
│   │   ├── templates/
│   │   │   ├── httpd_instance.conf.j2    # Per-instance httpd.conf
│   │   │   └── httpd_instance.service.j2 # Per-instance systemd unit
│   │   ├── handlers/main.yml       # Restart & systemd-reload handlers
│   │   └── meta/main.yml
│   └── haproxy/
│       ├── defaults/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── install_redhat.yml
│       │   └── install_debian.yml
│       ├── templates/
│       │   └── haproxy.cfg.j2      # Full HAProxy config (frontend + backend pool)
│       ├── handlers/main.yml
│       └── meta/main.yml
```

---

## Variables Reference

### `multi_httpd` role

| Variable | Default | Description |
|---|---|---|
| `httpd_instance_count` | `3` | Number of independent httpd instances to create. Can be 2–15+ |
| `httpd_base_port` | `8080` | First instance listen port; each subsequent instance adds 1 |
| `httpd_config_root` | `/etc/httpd/instances` | Parent dir for per-instance config dirs |
| `httpd_log_root` | `/var/log/httpd/instances` | Parent dir for per-instance log dirs |
| `httpd_doc_root` | `/var/www/html` | DocumentRoot for all instances |
| `httpd_server_admin` | `webmaster@localhost` | ServerAdmin email |
| `httpd_mpm_module` | `mpm_event` | MPM module: `mpm_event` or `mpm_prefork` |
| `httpd_start_servers` | `2` | Initial child processes/threads |
| `httpd_min_spare_threads` | `25` | Minimum idle threads |
| `httpd_max_spare_threads` | `75` | Maximum idle threads |
| `httpd_threads_per_child` | `25` | Threads per child process |
| `httpd_max_request_workers` | `150` | Max simultaneous requests **per instance** |
| `httpd_max_connections_per_child` | `0` | 0 = unlimited |
| `httpd_keepalive` | `On` | Enable persistent connections |
| `httpd_keepalive_timeout` | `5` | Seconds to wait for next keep-alive request |
| `httpd_max_keepalive_requests` | `100` | Max requests per keep-alive connection |
| `httpd_timeout` | `300` | Request timeout in seconds |

### `haproxy` role

| Variable | Default | Description |
|---|---|---|
| `haproxy_frontend_port` | `443` | HTTPS listener port |
| `haproxy_ssl_cert` | `/etc/ssl/certs/server.pem` | Combined PEM (cert chain + key) |
| `haproxy_balance` | `roundrobin` | Algorithm: `roundrobin`, `leastconn`, `source`, `random` |
| `haproxy_health_check_interval` | `2000` | ms between health checks |
| `haproxy_health_check_fall` | `3` | Failures before marking server DOWN |
| `haproxy_health_check_rise` | `2` | Successes before marking server UP |
| `haproxy_timeout_connect` | `5s` | Backend connect timeout |
| `haproxy_timeout_client` | `30s` | Client inactivity timeout |
| `haproxy_timeout_server` | `30s` | Backend response timeout |
| `haproxy_maxconn` | `50000` | Global max connections |
| `haproxy_stats_enabled` | `false` | Enable the stats dashboard (disabled by default) |
| `haproxy_stats_port` | `8404` | Stats listen port (loopback only) |
| `haproxy_stats_uri` | `/stats` | Stats page URI |
| `haproxy_stats_user` | `""` | Stats basic-auth username — **required** when stats enabled (asserted) |
| `haproxy_stats_password` | `""` | Stats basic-auth password — **required** when stats enabled (asserted) |
| `haproxy_stats_admin` | `false` | Enable stats admin mode (allows runtime changes) |
| `haproxy_http_redirect` | `true` | Redirect HTTP → HTTPS |
| `haproxy_http_port` | `80` | HTTP listener port (for redirect) |

---

## OS Support

| Family | Tested Distributions |
|---|---|
| **RedHat** | RHEL 7/8/9 · CentOS 7 · Rocky Linux 8/9 · AlmaLinux 8/9 |
| **Debian** | Ubuntu 20.04 / 22.04 / 24.04 · Debian 11 / 12 |

The playbook uses `ansible_os_family` to select the correct package manager and sets OS-specific facts (`httpd_binary`, `httpd_run_user`, `httpd_run_group`, `httpd_modules_dir`) automatically.

---

## TLS Certificate

HAProxy expects a **combined PEM file** containing the full certificate chain followed immediately by the private key:

```bash
# Example: combine a Let's Encrypt cert with its key
cat /etc/letsencrypt/live/example.com/fullchain.pem \
    /etc/letsencrypt/live/example.com/privkey.pem \
    > /etc/ssl/certs/server.pem
chmod 600 /etc/ssl/certs/server.pem
```

Set `haproxy_ssl_cert` to the resulting file path.

---

## Scaling Up / Down

To change the number of instances at any time, re-run the playbook with the new count:

```bash
# Scale up to 10 instances
ansible-playbook -i inventory/hosts.yml site.yml -e "httpd_instance_count=10"

# Scale back down to 3
ansible-playbook -i inventory/hosts.yml site.yml -e "httpd_instance_count=3"
```

> **Note:** Scaling down does **not** automatically stop/disable removed service units. After reducing `httpd_instance_count`, manually stop any extra instances:
> ```bash
> systemctl stop httpd4 httpd5 ... httpdN
> systemctl disable httpd4 httpd5 ... httpdN
> ```

---

## Verifying the Deployment

```bash
# Check all instances are running
systemctl status httpd1 httpd2 httpd3

# Confirm each instance is listening
ss -tlnp | grep httpd

# Test a backend directly
curl -s http://127.0.0.1:8080/server-status | head

# Test through HAProxy (skip cert verification for self-signed certs)
curl -sk https://localhost/

# View HAProxy stats
curl -su admin:changeme http://localhost:8404/stats
```
