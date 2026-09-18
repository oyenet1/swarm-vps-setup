# Ansible — native alternative to `install.sh` / `setup.sh`

Does **everything** the shell scripts do, as idempotent Ansible tasks, without
calling them: Docker install → repo checkout → `.env` (gaps filled, secrets
generated once) → render all configs → UFW (**including Swarm ports** for
multi-node) → Swarm init → overlay network → build images → deploy stack →
PgBouncer auth → verify → summary.

Both paths share the same `/opt/infra/.env` with the same rules (existing
values always win), so switching between `sudo ./setup.sh -s 2222` and this
playbook — in either direction — is safe.

---

## 1. Requirements

| Where | What |
|---|---|
| Control machine (your laptop/CI) | `ansible-core` 2.15+ only. No collections needed. Install: `pipx install ansible-core` or `uv tool install ansible-core` |
| Target VPS | Apt-based Linux (Ubuntu/Debian), Python 3 (present by default), SSH access (root or a sudo user) |
| Network | SSH reachable; the play opens the rest itself |

## 2. Quick start — single VPS

```bash
cd ansible
cp inventory.ini.example inventory.ini
```

Edit `inventory.ini` — one line:

```ini
[vps_manager]
vps1 ansible_host=203.0.113.10 ansible_user=root ansible_port=22
```

Run:

```bash
ansible-playbook -i inventory.ini site.yml
```

What happens, in order (mirrors `setup.sh` step for step):

1. Installs `ca-certificates curl gnupg openssl git ufw`; installs Docker
   (via `get.docker.com` only if missing) + compose plugin; enables Docker.
2. Repairs the containerd snapshot dir if missing; clones/updates the repo
   to `/opt/infra` (`infra_repo_url` / `infra_branch`).
3. Creates `/opt/infra/.env` from `.env.example` **only if missing**, then
   fills gaps and generates any empty/placeholder secrets
   (`change_me_*`, `password`, `admin`, `admin123`). Your existing values
   are never touched — read them later with `cat /opt/infra/.env`.
4. Renders `pgbouncer.ini`, `userlist.generated.txt`, `haproxy.generated.cfg`,
   `servers.json`, `rclone.generated.conf`, `alertmanager.generated.yml`
   (plus self-signed PgBouncer cert when absent).
5. Configures UFW: deny incoming; allow your SSH port, PgBouncer
   (`6543`), Postgres direct (`5544`), **and Swarm ports
   `2377/tcp`, `7946/tcp+udp`, `4789/udp`** so workers can join later.
6. Inits Swarm as manager (auto-detects advertise address unless
   `infra_swarm_advertise_addr` is set), creates the attachable `infra`
   overlay network (fails loudly if it exists with the wrong driver/scope).
7. Builds `infra/postgres` / `infra/backup` images only when missing
   (skips the pg build for `postgis/*` / `postgres:*` overrides; uses a
   stock image for local-only backups when R2 is off — same as `setup.sh`).
8. Removes the old stack, waits for removal, redeploys
   (`docker stack deploy`), configures the `pgbouncer_auth` role +
   `get_auth()` function, then verifies PgBouncer (incl. a temp
   database+role probe proving wildcard routing) and Redis.
9. Prints connection URLs, ports, and where passwords live.

Re-running is safe: everything except the stack redeploy reports `ok`
when already correct. (Like `setup.sh`, a re-run redeploys the stack so
config changes take effect.)

## 3. Configuration — `group_vars/all.yml` or `-e`

All knobs live in `group_vars/all.yml` (defaults mirror `.env.example`).
Override per-host with `host_vars/<name>.yml` or ad-hoc:

```bash
# custom SSH port + presetting one secret, rest auto-generated
ansible-playbook -i inventory.ini site.yml \
  -e infra_ssh_port=2222 \
  -e infra_pgadmin_email=you@example.com \
  -e infra_postgres_password='use-a-vault-for-this'
```

Secrets behaviour (what you asked for):

- Leave `infra_*_password` **empty** → generated once (40-char alphanumeric,
  same alphabet as `setup.sh`), written to `.env`, preserved on re-runs.
  Look at them anytime: `sudo cat /opt/infra/.env`.
- Set a value → written to `.env` (overrides a placeholder, but an already
  real value you typed directly in `.env` is still respected — explicit
  `-e`/vars win only when non-empty).
- Prefer Vault? Put the passwords in Vault-encrypted `group_vars/all.yml`
  or `host_vars/` — the play treats them like any preset value.

Common scenarios:

```bash
### Setup mode — infra, panel, or both

`infra_mode` (default `"infra"`) mirrors `setup.sh --mode`:

```bash
# aaPanel server panel only (no Docker/Swarm/stack)
ansible-playbook -i inventory.ini site.yml -e infra_mode=panel
# aaPanel first, then the full infra stack
ansible-playbook -i inventory.ini site.yml -e infra_mode=both
```

| Mode | Does | Firewall adds |
|---|---|---|
| `infra` | Full flow from §2 | PgBouncer, Postgres direct, Swarm ports |
| `panel` | Official aaPanel installer (`install_panel_en.sh ipssl`), skipped if already installed; credentials → `/opt/infra/aapanel-install.log` | `7800`, `80`, `443` |
| `both` | Panel, then the full flow | all of the above |

Panel URL after install: `https://<server-ip>:7800`.

### Cloudflare R2 backups
ansible-playbook -i inventory.ini site.yml \
  -e infra_r2_backup_enabled=true \
  -e infra_r2_account_id=abc123 \
  -e infra_r2_access_key_id=... \
  -e infra_r2_secret_access_key=... \
  -e infra_r2_bucket=mybucket

# Email alerts
ansible-playbook -i inventory.ini site.yml \
  -e infra_alert_email_to=you@example.com \
  -e infra_smtp_user=sender@gmail.com \
  -e infra_smtp_password=app-password

# Boot persistence (systemd unit, same as systemd/install-service.sh)
ansible-playbook -i inventory.ini site.yml \
  -e infra_install_systemd_service=true

# Skip live verification (render + deploy only)
ansible-playbook -i inventory.ini site.yml -e infra_verify_stack=false

# Skip firewall changes (you manage UFW/firewalld yourself)
ansible-playbook -i inventory.ini site.yml -e infra_run_firewall=false
```

## 4. Multi-node — workers join the running Swarm

Workers run **no** infra services; they just need Docker, Swarm ports, and
the join token. Add them to `inventory.ini`:

```ini
[vps_workers]
worker1 ansible_host=203.0.113.11 ansible_user=root ansible_port=22
```

Then (manager must already be provisioned):

```bash
ansible-playbook -i inventory.ini join-workers.yml
```

The play fetches the worker token from the first `vps_manager` host,
opens `7946/tcp+udp` + `4789/udp` (+ SSH) on each worker, and runs
`docker swarm join`. Re-runs skip hosts already `active`. For extra
managers, promote a joined worker manually:
`docker node promote <node>` on the manager.

## 5. Switching between `.sh` and Ansible

Safe both ways — same files, same rules:

```bash
# provisioned with Ansible, now tweak via shell:
ssh root@vps "cd /opt/infra && ./setup.sh -s 2222"
# provisioned with shell, now manage via Ansible:
ansible-playbook -i inventory.ini site.yml
```

The only shared mutable state is `.env` + the generated files; neither tool
deletes values the other wrote.

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| `Network infra exists but is bridge/swarm` | A non-overlay network stole the name. `docker network rm infra` (or set `infra_network_name`) and re-run |
| Swarm init fails, port 2377 | UFW blocked it — re-run with `infra_run_firewall=true`, or `ufw allow 2377/tcp && ufw allow 7946/tcp && ufw allow 4789/udp` |
| `R2_BACKUP_ENABLED=true but … is empty` | Pass the R2 vars (see §3); same guard as `setup.sh` |
| PgBouncer probe / Redis wait fails | Services still converging — re-run; tasks wait ~60–120s before failing |
| `docker swarm join` refused on worker | Manager firewall or wrong manager IP; check `_manager_addr` (uses `infra_swarm_advertise_addr` when set, else `ansible_host`) |
| Template shows old values after editing `group_vars` | Remember: `.env` wins over defaults. Edit `/opt/infra/.env` on the host (or delete the key line) and re-run |

## 7. Files in this directory

| File | Purpose |
|---|---|
| `site.yml` | Manager provisioning (full `install.sh`+`setup.sh` mirror) |
| `join-workers.yml` | Swarm worker join |
| `inventory.ini.example` | Copy to `inventory.ini`, fill in hosts |
| `group_vars/all.yml` | All variables + defaults |
| `templates/` | `pgbouncer.ini`, `haproxy.cfg`, `servers.json`, `rclone.conf`, `alertmanager.yml`, systemd unit |
