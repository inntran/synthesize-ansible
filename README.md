Synthesize
==========

Installing Graphite doesn't have to be difficult. Synthesize provides Ansible roles to make it easy to install Graphite and related services on Debian 13 (Trixie).

> **Note:** This is a fork focused on Ansible-only deployment for Debian 13. Original project by Jason Dixon: https://github.com/obfuscurity/synthesize

## Version Support

**Debian 13 (Trixie) only** - uses native Debian packages (graphite-carbon 1.1.10-2 and graphite-web 1.1.10-8)

:warning: **WARNING:** You should not install Synthesize directly on your personal development system. It's strongly suggested that you use a VM or other temporary VPS instance for sandboxing Synthesize.

## Provides

* Graphite 1.1.x ([graphite-web](https://github.com/graphite-project/graphite-web), [carbon](https://github.com/graphite-project/carbon), [whisper](https://github.com/graphite-project/whisper))
* StatsD ([statsite](https://github.com/armon/statsite))
* [Collectd](http://collectd.org/)
* [Grafana](https://grafana.org/) (via official grafana.grafana collection)

The resulting Graphite web interface __listens only on HTTPS port 443__ and has been configured to collect metrics specifically for helping profile the performance of your Graphite and Carbon services. It uses memcached for improved query performance, and Statsite for a fast, C-based implementation of the StatsD collector/aggregator.

Grafana provides a modern and full-featured alternative to Graphite's built-in Composer and Dashboard interfaces. It also includes a default dashboard for monitoring Carbon's internal statistics.

## Installation

### Prerequisites

* Ansible 2.9 or later
* A minimal Debian 13 (Trixie) installation
* SSH access with sudo privileges

### Quick Start

1. Clone this repository:
```bash
git clone https://github.com/obfuscurity/synthesize.git
cd synthesize
```

2. Install required Ansible collections:
```bash
ansible-galaxy collection install grafana.grafana
```

3. Create your inventory file:
```bash
cp inventory.example inventory
# Edit inventory to add your Debian 13 server(s)
```

4. Run the playbook:
```bash
ansible-playbook -i inventory playbook.yml
```

For a local installation (testing):
```bash
ansible-playbook -i localhost, -c local playbook.yml
```

### Customization

You can customize the installation by overriding variables in your playbook or inventory:

```yaml
- hosts: graphite_servers
  become: true
  roles:
    - debian_base
    - graphite
    - apache_ssl
    - grafana.grafana.grafana
  vars:
    # Graphite configuration
    carbon_instances: 2          # Run multiple carbon-cache instances
    statsite_enabled: true       # Enable/disable Statsite (default: true)
    collectd_enabled: true       # Enable/disable Collectd (default: true)

    # Grafana configuration (collection-specific variables)
    grafana_domain: "{{ ansible_fqdn }}"
    grafana_url: "https://{{ ansible_fqdn }}/grafana/"
    grafana_security:
      admin_user: admin
      admin_password: secure_password
    grafana_server:
      serve_from_sub_path: true
```

See role defaults in:
* `roles/debian_base/defaults/main.yml`
* `roles/graphite/defaults/main.yml`
* `roles/apache_ssl/defaults/main.yml`

## Role Structure

Synthesize uses a modular role structure:

1. **debian_base** - Validates Debian 13 and installs base packages
2. **graphite** - Installs and configures Graphite, Carbon, Statsite, and Collectd
3. **apache_ssl** - Configures Apache with SSL as a reverse proxy
4. **grafana.grafana.grafana** - Official Grafana collection for installation

## Post-Installation

### Access the Services

* **Landing Page:** https://your-server/ (HTTPS port 443)
* **Graphite Web UI:** https://your-server/graphite/
* **Grafana:** https://your-server/grafana/
* **Carbon line receiver:** Port 2003 (plaintext metrics)
* **Carbon pickle receiver:** Port 2004 (binary metrics)
* **Statsite (StatsD):** Port 8125 (UDP/TCP)

All web interfaces are served through Apache with SSL.

### Default Credentials

#### Graphite-Web

* username: `admin`
* password: `graphite_me_synthesize`

Change the password:
```bash
graphite-manage changepassword admin
```

#### Grafana

* username: `admin`
* password: `admin`

You'll be prompted to change the password on first login.

## Managing Services

```bash
# Carbon cache
systemctl start|stop|restart carbon-cache
systemctl status carbon-cache

# Other services
systemctl start|stop|restart statsite
systemctl start|stop|restart grafana-server
systemctl start|stop|restart apache2
systemctl start|stop|restart collectd
systemctl start|stop|restart memcached

# Check graphite index rebuild timer
systemctl status graphite-build-index.timer
```

## Upgrading

Re-run the playbook with updated package versions. Debian package updates will be handled automatically:

```bash
ansible-playbook -i inventory playbook.yml
```

## Removal

To completely remove the installation:

```bash
# Stop and disable services
systemctl stop carbon-cache statsite grafana-server apache2 collectd
systemctl disable carbon-cache statsite grafana-server

# Remove packages
apt-get purge graphite-web graphite-carbon python3-whisper grafana

# Remove data directories (CAUTION: This deletes all metrics!)
rm -rf /var/lib/graphite
rm -rf /var/lib/grafana
```

## Architecture

The Ansible roles use:
* Native Debian packages for Graphite components (no building from source)
* Systemd for service management
* Apache with mod_wsgi for serving Graphite-web
* Memcached for query performance
* Statsite built from source (not packaged in Debian)
* Grafana from official Grafana APT repository via grafana.grafana collection

See `CLAUDE.md` for detailed architecture documentation.

## Troubleshooting

### Check service status
```bash
systemctl status carbon-cache
systemctl status apache2
journalctl -u carbon-cache -f
```

### Verify Graphite is receiving metrics
```bash
echo "test.metric 42 `date +%s`" | nc localhost 2003
```

### Check Apache logs
```bash
tail -f /var/log/apache2/graphite-error.log
tail -f /var/log/apache2/graphite-access.log
```

### Check Graphite-web logs
```bash
tail -f /var/log/graphite-web/info.log
tail -f /var/log/graphite-web/exception.log
```

## License

This set of Ansible roles and playbooks are distributed under the MIT license.
