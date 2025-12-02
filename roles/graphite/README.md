# Graphite Ansible Role

This Ansible role installs and configures Graphite, Carbon, Statsite, and Collectd on Debian 13 (Trixie).

## Requirements

* Debian 13 (Trixie)
* Ansible 2.9 or later
* Root or sudo access

## Role Variables

All variables are defined in `defaults/main.yml` with sensible defaults. Key variables:

### Paths (Debian FHS-compliant)
```yaml
graphite_home: /usr/share/graphite-web   # Web application files
graphite_conf: /etc/carbon               # Carbon configuration
graphite_storage: /var/lib/graphite      # Whisper data storage
graphite_log_dir: /var/log/carbon        # Carbon logs
graphite_web_log_dir: /var/log/graphite-web  # Web app logs
```

### Carbon Configuration
```yaml
carbon_instances: 1           # Number of carbon-cache instances
carbon_user: _graphite        # System user (created by Debian package)
carbon_group: _graphite       # System group (created by Debian package)
```

### Feature Toggles
```yaml
statsite_enabled: true        # Install Statsite (StatsD)
collectd_enabled: true        # Install and configure Collectd
```

### Django Admin
```yaml
django_admin_user: admin
django_admin_password: graphite_me_synthesize
django_admin_email: admin@localhost
```

### Statsite Configuration
```yaml
statsite_release: master      # Git branch/tag to build
statsite_repo: https://github.com/armon/statsite.git
```

## Dependencies

Depends on `debian_base` role for base package installation.

## Example Playbook

### Basic Installation
```yaml
- hosts: graphite_servers
  become: true
  roles:
    - debian_base
    - graphite
    - apache_ssl
```

### Customized Installation
```yaml
- hosts: graphite_servers
  become: true
  roles:
    - role: graphite
      vars:
        carbon_instances: 3
        statsite_enabled: true
        collectd_enabled: true
```

## Post-Installation

### Service Management

Carbon cache:
```bash
systemctl start|stop|restart carbon-cache
systemctl status carbon-cache
```

Other services:
```bash
systemctl status statsite
systemctl status collectd
systemctl status graphite-build-index.timer
```

### Access Points

* Carbon line receiver: port 2003 (plaintext)
* Carbon pickle receiver: port 2004 (binary)
* Statsite: port 8125 (UDP/TCP for StatsD protocol)

### Default Credentials

**Graphite:**
* Username: admin
* Password: graphite_me_synthesize

Change the password:
```bash
graphite-manage changepassword admin
```

## Architecture

The role performs these steps:

1. Installs Graphite packages from Debian repos (graphite-web, graphite-carbon, python3-whisper)
2. Builds and installs Statsite from source (if enabled)
3. Installs Collectd for metrics collection (if enabled)
4. Generates Django SECRET_KEY and saves to /etc/graphite/.secret_key
5. Configures Carbon, Graphite-web, and Collectd
6. Runs Django migrations and creates superuser
7. Sets up systemd services for Carbon and Statsite
8. Configures hourly index rebuild via systemd timer
9. Enables and starts all services

## Files

### Templates (Jinja2)
* `carbon.conf.j2` - Carbon configuration
* `storage-schemas.conf.j2` - Whisper storage schemas
* `storage-aggregation.conf.j2` - Whisper aggregation rules
* `local_settings.py.j2` - Graphite-web Django settings
* `collectd.conf.j2` - Collectd configuration
* `statsite.conf.j2` - Statsite configuration

### Static Files
* `initial_data.json` - Django initial data
* `graphite-build-index` - Index rebuild script
* `graphite-build-index.service` - Index rebuild service
* `graphite-build-index.timer` - Hourly timer
* `statsite.service` - Statsite systemd service
* `GraphiteCarbonMetrics_obfuscurity.json` - Grafana dashboard

## License

MIT

## Author

Yinchuan Song (Red Hat)
