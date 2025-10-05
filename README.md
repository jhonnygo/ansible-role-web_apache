# Ansible Role: **web_apache**
[![Style: ansible-lint](https://img.shields.io/badge/style-ansible--lint-green)](#lint)
[![Tests: molecule](https://img.shields.io/badge/tests-molecule-blue)](#testing)
[![License: MIT](https://img.shields.io/badge/license-MIT-informational)](LICENSE)

Ansible role to **install and configure Apache2 with declarative vhosts** on Debian/Ubuntu:

- Installs `apache2` and manages its **service state** (`started`/`stopped`) and **enablement** on boot.
- Declarative **VirtualHosts** via variables; renders from a **Jinja2 template** (overrideable per vhost).
- Creates **docroots** (0755), optional **`index.php` test file**, and per‑vhost **access/error logs**.
- Enables sites with `a2ensite`, optionally **disables** `000-default.conf`, and **reloads** Apache on changes.
- Optional **catch‑all** site (modes `403`, `404`, or `redirect`) to control unmatched hosts.
- Optional **purge of unmanaged vhosts** (safe excludes), and optional **docroot deletion** under allowed prefixes.
- Idempotent; supports **check mode** for most tasks.
- Includes **Molecule** (Ubuntu Jammy) to validate package install, port 80 listening and a vhost responding.
- Ready for **Ansible Galaxy** as a standalone role (`v0.1.2`) with `galaxy.yml`/`meta/main.yml` in place.

---

## Table of Contents
- [Repository Layout (How I run it)](#repository-layout-how-i-run-it)
- [Compatibility & Requirements](#compatibility--requirements)
- [Installation](#installation)
  - [Example `collections/requirements.yml`](#example-collectionsrequirementsyml)
- [Inventory & Useful Commands](#inventory--useful-commands)
- [Variables](#variables)
  - [Structure of `web_apache_vhosts`](#structure-of-web_apache_vhosts)
  - [Control variables & defaults](#control-variables--defaults)
- [Usage Examples](#usage-examples)
- [Tags](#tags)
- [Idempotence & Safety](#idempotence--safety)
- [Troubleshooting](#troubleshooting)
- [Testing](#testing)
- [Lint](#lint)
- [Publishing to Galaxy](#publishing-to-galaxy)
- [License & Author](#license--author)
- [Test Matrix / Verified Environments](#test-matrix--verified-environments)

---

## Repository Layout (How I run it)

Below is the **project layout I currently use**. It matches the layout in my other roles, so the command examples reuse the same paths.

```
.
├── ansible.cfg
├── collections/
│   ├── ansible_collections/
│   └── requirements.yml
├── inventories/
│   ├── prod/
│   │   ├── aws_ec2.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── role_app.yml
│   └── stage/
│       ├── aws_ec2.yml
│       └── group_vars/
│           ├── all.yml
│           └── role_app.yml
├── playbooks/
│   ├── web_apache.yml
│   └── ping.yml
└── roles/
    └── jhonnygo.web_apache/
        ├── CHANGELOG
        ├── LICENSE
        ├── README.md   # this file
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── meta/
        │   └── main.yml
        ├── tasks/
        │   ├── vhosts.yml
        │   └── main.yml
        ├── templates/
        │   ├── vhost.conf.j2
        │   ├── default.conf.j2
        │   ├── default-ssl.j2
        │   └── catchall.conf.j2
        └── molecule/
            └── default/
                ├── converge.yml
                ├── verify.yml
                └── molecule.yml
```

> **Note**  
> I use **dynamic inventory** with `aws_ec2.yml` inside `inventories/<env>/` and group variables under `group_vars/`. The role usually targets `role_app`.

---

## Compatibility & Requirements

**Systems**: Debian/Ubuntu (Apache 2.4).  
**Ansible Collections**: Only `ansible.builtin` is required. You may still pin common collections:

```yaml
# collections/requirements.yml
collections:
  - name: ansible.posix
  - name: community.general
```

Install them with:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

---

## Installation

You can consume the role in three ways:

### 1) Using Git directly (one‑off)
```bash
ansible-galaxy role install git+https://github.com/jhonnygo/ansible-role-web_apache.git
```

### 2) Using a unified *requirements.yml* (recommended)
Keep a single **`collections/requirements.yml`** that includes **both collections and roles**. Then install each with its command:

```yaml
# collections/requirements.yml
---
collections:
  - name: amazon.aws
  - name: ansible.posix
  - name: community.general

roles:
  - name: jhonnygo.web_apache
    src: https://github.com/jhonnygo/ansible-role-web_apache.git
    version: "0.1.2"
```

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-galaxy role       install -r collections/requirements.yml
```

### 3) Ansible Galaxy
```bash
ansible-galaxy role install jhonnygo.web_apache
```

---

## Inventory & Useful Commands

Graph dynamic inventory:
```bash
ansible-inventory -i inventories/stage/aws_ec2.yml --graph
```

Dry run (only vhosts):
```bash
ansible-playbook -i inventories/stage/aws_ec2.yml playbooks/web_apache.yml \
  -l role_app --tags web_apache:vhosts --check --diff
```

Run for real:
```bash
ansible-playbook -i inventories/stage/aws_ec2.yml playbooks/web_apache.yml -l role_app
```

---

## Variables

### Structure of `web_apache_vhosts`

```yaml
web_apache_vhosts:
  - server_name: "stage.jhonnygo.com"
    docroot: "/var/www/cv"
    create_docroot: true
    create_index_php: true
    index_php_content: "<?php echo 'CV STAGE'; ?>"
    # Optional:
    server_alias: "www.stage.jhonnygo.com"
    extra_directives: |
      <Directory /var/www/cv>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
      </Directory>
    access_log:  "/var/log/apache2/cv-access.log"
    error_log:   "/var/log/apache2/cv-error.log"
    template: "vhost.conf.j2"

  - server_name: "stage.yotkt.com"
    docroot: "/var/www/api"
    create_docroot: true
    create_index_php: true
    index_php_content: "<?php echo 'API STAGE'; ?>"
```

### Control variables & defaults

```yaml
# defaults/main.yml
web_apache_packages:
  - apache2

web_apache_state: started        # started | stopped
web_apache_enabled: true
web_apache_listen_port: 80

# Default HTTP
web_apache_manage_default_site: true
web_apache_remove_default_site: false
web_apache_default_filename: "000-default.conf"
web_apache_overwrite_defaults: false
web_apache_restore_defaults_if_missing: true

# Default SSL
web_apache_manage_default_ssl: true
web_apache_remove_default_ssl: false
web_apache_default_ssl_filename: "default-ssl.conf"

# Catch‑all
web_apache_manage_catch_all: false
web_apache_catch_all_filename: "000-catch-all.conf"
web_apache_catch_all_mode: "403"            # 403 | 404 | redirect
web_apache_catch_all_redirect_to: "https://example.com"
web_apache_catch_all_docroot: "/var/www/empty"

# Purge
web_apache_purge_unlisted_vhosts: false
web_apache_purge_exclude_names: ["000-default","default-ssl","000-catch-all"]
web_apache_purge_docroots: false
web_apache_allowed_docroot_prefixes: ["/var/www"]

# Declarative vhosts list
web_apache_vhosts: []
```

---

## Usage Examples

### Minimal play
```yaml
- name: Install and configure Apache + vhosts (stage)
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: web_apache
      vars:
        web_apache_vhosts:
          - server_name: "stage.jhonnygo.com"
            docroot: "/var/www/cv"
            create_docroot: true
            create_index_php: true
            index_php_content: "<?php echo 'CV STAGE'; ?>"
          - server_name: "stage.yotkt.com"
            docroot: "/var/www/api"
            create_docroot: true
            create_index_php: true
            index_php_content: "<?php echo 'API STAGE'; ?>"
```

### Keep defaults enabled + add catch‑all
```yaml
web_apache_manage_default_site: true
web_apache_remove_default_site: false
web_apache_manage_default_ssl: true
web_apache_remove_default_ssl: false

web_apache_manage_catch_all: true
web_apache_catch_all_mode: "redirect"
web_apache_catch_all_redirect_to: "https://stage.sientetuser.com"
```

### Purge unmanaged vhosts (and their docroots under /var/www)
```yaml
web_apache_purge_unlisted_vhosts: true
web_apache_purge_docroots: true
web_apache_allowed_docroot_prefixes:
  - "/var/www"
web_apache_purge_exclude_names:
  - "000-default"
  - "default-ssl"
  - "000-catch-all"
```

---

## Tags

- `web_apache` — general role tag  
- `web_apache:install` — package installation  
- `web_apache:service` — service ensure/reload  
- `web_apache:vhosts` — vhost rendering/enabling/purge  
- `vhosts` — shorthand

Run only vhosts workflow:
```bash
ansible-playbook -i inventories/stage/aws_ec2.yml playbooks/web_apache.yml \
  -t "web_apache:vhosts" --check --diff
```

---

## Idempotence & Safety

- `a2ensite/a2dissite` calls are guarded with `creates`/`removes` for idempotence.
- Critical symlinks for stock sites and optional catch‑all are ensured as **symlinks** (no stray files).
- Purge keeps stock files by default via `web_apache_purge_exclude_names`.
- Docroot deletion is confined to **safe prefixes** only (default `/var/www`).

---

## Troubleshooting

**Default site is not enabled**  
Ensure:
- `web_apache_manage_default_site: true`
- `web_apache_remove_default_site: false`
- `web_apache_default_filename: "000-default.conf"`

**Wrong vhost answers an unmatched host**  
Apache serves the first loaded site for unmatched hosts. Keep **000-default** enabled or configure a **catch‑all**.

**Using AWS SSM**  
If you see a spurious `Exception ignored in: <function Connection.__del__ ...>` at teardown, add a final `meta: flush_handlers` + a short pause in your play, or filter stderr in a wrapper (see notes in issues).

---

## Testing

Molecule (Ubuntu Jammy) validates:
- `apache2` package present
- `apache2` service active
- Port 80 listening
- A vhost responds on `/`

```bash
pip install molecule molecule-plugins[docker] ansible-lint
cd roles/jhonnygo.web_apache/molecule/default
molecule test
```

---

## Lint

```bash
ansible-lint roles/jhonnygo.web_apache
yamllint roles/jhonnygo.web_apache
```

---

## Galaxy

Repo: `github.com/jhonnygo/ansible-role-web_apache`

`collections/requirements.yml` example:
```yaml
roles:
  - name: jhonnygo.web_apache
    version: "0.1.2"
```

```bash
# install (consumers)
ansible-galaxy role install jhonnygo.web_apache
```

---

<br/>

<img src="img/happy-coding.jpg?raw=true" alt="Footer Logo" />

## License & Author

**License:** MIT  
**Author:** Jhonny Alexander (JhonnyGO) — <support@jhoncytech.com>  
**GitHub:** https://github.com/jhonnygo/ansible-role-web_apache
