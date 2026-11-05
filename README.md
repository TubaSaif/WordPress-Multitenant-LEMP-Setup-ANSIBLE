# WordPress multi-tenant LEMP VPS (Ansible)

One Ubuntu 24.04 VPS, many customers ("tenants"). Each tenant can own any number
of WordPress sites - plain single installs **or** multisite networks
(subdomain / subdirectory).

## How it runs (remote / control-node model)

```
 Control node (laptop / CI-WINDOWS BASED USE WSL)  --SSH-->  Managed node (the VPS)
 ansible-playbook lives here           needs only Python 3 + sshd
```
Ansible pushes small modules over SSH, runs them, and removes them. Nothing but
Python is required on the VPS.

## Isolation model (per tenant)

| Layer        | What the tenant gets                                                     |
|--------------|--------------------------------------------------------------------------|
| Linux user   | `wp_<name>`, no shell, **SFTP-only, chrooted** to `/srv/tenants/<name>`   |
| PHP          | Own **PHP-FPM pool** running as that user, `open_basedir` jail, private tmp, `ondemand` workers |
| Database     | One DB + one MariaDB user per site, privileges on that DB only            |
| Files        | `/srv/tenants/<name>/sites/<domain>/public`, unreadable by other tenants  |
| Cron         | System cron per site replaces WP-Cron (multisite: iterates all sub-sites) |
| TLS          | Let's Encrypt per site (HTTP-01), or DNS-01 wildcard for multisite subdomains |
| Logs         | `/var/log/nginx/tenants/<name>/` (root-owned), rotated; fail2ban watches wp-login |

Server-wide: UFW (22/80/443), fail2ban, unattended security upgrades,
nginx catch-all that drops unknown hostnames, nightly local backups.


## Quick start

### 1. Set up the control node

Copy this repo in control node.
Run this on your laptop or any machine that will run Ansible (not on the VPS(Managed Node)):

```bash
sudo apt update
sudo apt install -y pipx
pipx ensurepath
pipx install ansible-core
ansible-galaxy collection install -r requirements.yml
```
Open a new terminal after `pipx ensurepath` so `ansible` is on your PATH.

### 2. Configure

1. **Inventory:** edit `inventory/production.yml` and point it at your VPS. Use root or a sudo user with SSH key authentication.
2. **Settings:** edit `group_vars/wordpress/main.yml` (`letsencrypt_email`, `php_version`, server sizing).
3. **Tenants:** edit `group_vars/wordpress/tenants.yml` (your tenants and their sites).
4. **DNS:** create A/AAAA records for every site domain pointing to the VPS IP. This must be done **before** the first run, or Let's Encrypt certificate issuance will fail. - For Local Practice Setup:  Set SSL:false

### 3. Check connectivity

```bash
ansible -i inventory/production.yml wordpress -m ping
```

### 4. Dry run

```bash
ansible-playbook -i inventory/production.yml site.yml --check --diff
```

Some steps are skipped or reported inaccurately in check mode (for example, tasks that depend on packages or files that don't exist yet), so a clean dry run is a sanity check, not a guarantee.

### 5. Provision

```bash
ansible-playbook -i inventory/production.yml site.yml
```

Add `--ask-become-pass` if your SSH user needs a sudo password.


### Day-2 operations

```bash
# add / change tenants or sites (fast; touches only the tenants role)
ansible-playbook site.yml --tags tenants

# only re-render PHP / nginx / DB config
ansible-playbook site.yml --tags php,nginx,db
```

* **Add a tenant**: append to `tenants:` and run `--tags tenants`.
* **Add a site to a tenant**: append to that tenant's `sites:`; same command.
* **Single vs multisite**: `type: single` (default) or `type: multisite` with
  `multisite_mode: subdomain|subdirectory`.
* **DNS not ready yet?** set `ssl: false` on the site, deploy, then flip to `true`
  once DNS resolves (certbot's HTTP-01 check needs it).
* **Multisite + subdomains** needs a wildcard cert: set `wildcard_ssl: true`, add a
  Cloudflare token to `group_vars/wordpress/vault.yml` (`ansible-vault encrypt` it),
  and run with `--ask-vault-pass`. (Other DNS providers: swap the certbot plugin.)
* **Tenant login**: `sftp wp_alice@your-vps` (key auth only). They see
  `sites/` and `tmp/` inside their jail. No shell.

## Credentials

Generated on first run and stored **on the control node** in `credentials/<domain>/`
(`db_password`, `wp_admin_password`). They're gitignored; back them up / encrypt
them. Re-running never regenerates them. WP admin username defaults to the tenant name.

## For Local WSL Setup- With DUmmy domains
Add VPS IP(Managed node) with domain name in windows host file in order to access site via browser.

Note: For Local Practice Setup, no cloudflare or certbort ssl, so set ssl:false.