# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Synthesize is an automated installation system for Graphite and related monitoring tools. It provides two installation methods:

1. **Ansible Role (Recommended):** For Debian 13 (Trixie) using native Debian packages
2. **Legacy Shell Script:** For Ubuntu 18.04 LTS building from source

**Core Purpose:** Provide a turnkey Graphite monitoring stack installation (Graphite-web, Carbon, Whisper, Statsite, Collectd, and Grafana) with production-ready defaults.

**Platform Support:**
- Ansible role: Debian 13 (Trixie) only - uses packaged graphite-carbon 1.1.10-2 and graphite-web 1.1.10-8
- Shell script: Ubuntu 18.04 LTS only - builds from source (deprecated)

## Architecture

### Ansible Role Architecture (Debian 13)

The Ansible role is located in `roles/graphite/` and follows standard Ansible Galaxy structure:

**Directory Structure:**
- `tasks/` - Main installation logic split into main.yml, statsite.yml, and grafana.yml
- `templates/` - Jinja2 templates for all configuration files
- `files/` - Static files (systemd units, scripts, Grafana dashboards)
- `defaults/` - Default variables in main.yml
- `handlers/` - Service restart handlers
- `meta/` - Role metadata

**Installation Flow:**
1. Validates Debian 13 (Trixie) - fails if not correct OS
2. Checks if Graphite already installed (fails if /opt/graphite exists)
3. Installs Graphite packages from Debian repos (no source building)
4. Builds Statsite from source (only component not packaged)
5. Installs Grafana from official Grafana APT repository
6. Creates carbon system user/group (_graphite:_graphite)
7. Templates all configuration files with Jinja2
8. Runs Django migrations and collects static files
9. Configures Apache with mod_wsgi and SSL
10. Enables and starts all systemd services

**Key Differences from Shell Script:**
- Uses native Debian packages instead of building from Git
- Idempotent - can be run multiple times safely
- Variables in defaults/main.yml allow easy customization
- Proper Ansible handlers for service management
- No hard-coded paths or values in templates

### Legacy Shell Script Architecture (Ubuntu 18.04)

1. **install script** - Main entry point that:
   - Validates Ubuntu 18.04
   - Installs apt dependencies (Apache, Python 3, build tools, memcached, collectd)
   - Clones Graphite projects from GitHub (graphite-web, carbon, whisper)
   - Builds and installs components to `/opt/graphite`
   - Configures services via systemd units
   - Sets up Apache with SSL (HTTPS on port 443 only)
   - Installs and configures Grafana with provisioned datasources/dashboards

2. **Templates directory** - Contains all configuration files that are copied during installation:
   - `graphite/conf/` - Carbon configuration (carbon.conf, storage schemas/aggregation)
   - `graphite/webapp/` - Django settings (local_settings.py, initial_data.json)
   - `apache/` - Apache virtual host with SSL configuration
   - `systemd/` - Service units for carbon-cache, statsite, graphite-build-index
   - `scripts/` - Control scripts (carbon-cache wrapper, index builder)
   - `collectd/` - Metrics collection configuration
   - `statsite/` - StatsD-compatible aggregator configuration
   - `grafana/` - Provisioned datasources and dashboards

3. **Service Management**
   - Carbon-cache runs as systemd service instances (`carbon-cache@1.service`)
   - Control script at `/usr/local/bin/carbon-cache` manages multiple instances
   - Default is 1 instance, configurable via INSTANCES variable in the script
   - Hourly systemd timer runs `graphite-build-index` to update search index

### Key Paths

- **Graphite Home:** `/opt/graphite`
- **Graphite Config:** `/opt/graphite/conf`
- **Graphite Storage:** `/opt/graphite/storage`
- **Whisper Data:** `/opt/graphite/storage/whisper`
- **Control Scripts:** `/usr/local/bin/` (carbon-cache, graphite-build-index)
- **Statsite Binary:** `/usr/local/sbin/statsite`
- **Source Repositories:** `/usr/local/src/` (graphite-web, carbon, whisper, statsite)

### Networking

The Vagrantfile configures these port forwards (host:guest):
- 8443:443 - Graphite-web HTTPS interface
- 8125:8125 (TCP/UDP) - Statsite (StatsD protocol)
- 22003:2003 - Carbon line receiver (plaintext)
- 22004:2004 - Carbon pickle receiver (binary)
- 3030:3000 - Grafana web interface

## Common Commands

### Ansible Installation (Debian 13)

```bash
# Run playbook against inventory
ansible-playbook -i inventory playbook.yml

# Local installation for testing
ansible-playbook -i localhost, -c local playbook.yml

# Run with custom variables
ansible-playbook -i inventory playbook.yml -e "carbon_instances=3"

# Check what would change (dry run)
ansible-playbook -i inventory playbook.yml --check --diff

# Run only specific tags (if implemented)
ansible-playbook -i inventory playbook.yml --tags grafana

# Verbose output for debugging
ansible-playbook -i inventory playbook.yml -vvv
```

### Legacy Installation & Management (Ubuntu 18.04)

```bash
# Install Graphite stack
sudo ./install

# Upgrade existing installation (experimental, requires DANGER_ZONE=TRUE)
DANGER_ZONE=TRUE sudo ./upgrade

# Uninstall completely
sudo ./uninstall
```

### Carbon Cache Control

```bash
# Enable carbon-cache instances at boot
/usr/local/bin/carbon-cache enable

# Start all carbon-cache instances
/usr/local/bin/carbon-cache start

# Stop all instances
/usr/local/bin/carbon-cache stop

# Restart all instances
/usr/local/bin/carbon-cache restart

# Check status
systemctl status carbon-cache@1
```

### Service Management

```bash
# Start/stop individual services
systemctl start|stop|restart statsite
systemctl start|stop|restart grafana-server
systemctl start|stop|restart apache2
systemctl start|stop|restart memcached
systemctl start|stop|restart collectd

# Check graphite-build-index timer
systemctl status graphite-build-index.timer
systemctl status graphite-build-index.service
```

### Vagrant Development

```bash
# Provision VM
vagrant plugin install vagrant-vbguest
vagrant up

# Destroy VM
vagrant destroy

# Set custom Graphite version during provisioning
GRAPHITE_RELEASE=1.1.7 vagrant up
```

### Django Admin

```bash
# Create Graphite superuser (Debian 13 with packaged graphite-web)
graphite-manage createsuperuser
# Note: You may see a warning about STATICFILES_DIRS, this is normal and can be ignored

# Change Graphite admin password
cd /opt/graphite/webapp/graphite
sudo python3 manage.py changepassword admin

# Run Django migrations
PYTHONPATH=/opt/graphite/webapp django-admin.py migrate --settings=graphite.settings

# Collect static files
PYTHONPATH=/opt/graphite/webapp django-admin.py collectstatic --settings=graphite.settings
```

## Development Notes

### Carbon Cache Instances

The carbon-cache control script supports multiple instances through systemd templates. To add more instances:

1. Edit `/usr/local/bin/carbon-cache` and change `INSTANCES=1` to desired number
2. Add corresponding sections in `/opt/graphite/conf/carbon.conf` (e.g., `[cache:2]`, `[cache:3]`)
3. Run `/usr/local/bin/carbon-cache enable` to enable new instances

### Configuration Files

When modifying templates:
- Changes only take effect on new installations or upgrades
- The `upgrade` script backs up configs before overwriting (adds `.backup` suffix)
- `install` script performs one-time substitution: `UNSAFE_DEFAULT` → random MD5 hash in local_settings.py

### Python Environment

- Uses Python 3 throughout (python3, pip3)
- Django version pinned to 2.2.9 for compatibility
- Graphite version controlled by `GRAPHITE_RELEASE` environment variable (default: 1.1.7)
- All Python packages installed globally, not in virtualenv

### Security Considerations

- Apache configured for HTTPS only (no HTTP on port 80)
- Uses self-signed SSL certificate
- Default credentials: admin/graphite_me_synthesize (Graphite), admin/admin (Grafana)
- Carbon unpickler security: `USE_INSECURE_UNPICKLER = False` in carbon.conf
- Carbon user runs as restricted system user (uid 998, no shell)

### Storage Schemas

Default storage schema captures high-resolution data and aggregates over time. Modify `/opt/graphite/conf/storage-schemas.conf` before installation or restart carbon-cache after changes.

### Known Issues

- Windows users may need to run `dos2unix` on install script before provisioning
- Upgrade script is experimental and requires explicit `DANGER_ZONE=TRUE` acknowledgment
- Tags feature disabled in carbon.conf due to SSL compatibility issues
