# apache_ssl

Apache web server with SSL configuration for Graphite and Grafana reverse proxy.

## Requirements

- Debian 13 (Trixie)
- Graphite installed (for WSGI configuration)

## Role Variables

See `defaults/main.yml` for all available variables:

- `apache_packages`: List of Apache packages to install
- `apache_ssl_cert`: Path to SSL certificate (default: snakeoil cert)
- `apache_ssl_key`: Path to SSL private key (default: snakeoil key)
- `carbon_user`: User for WSGI daemon process (default: _graphite)
- `carbon_group`: Group for WSGI daemon process (default: _graphite)

## Dependencies

None (but expects Graphite to be installed for WSGI configuration)

## Example Playbook

```yaml
- hosts: servers
  become: true
  roles:
    - apache_ssl
  vars:
    apache_ssl_cert: /path/to/your/cert.pem
    apache_ssl_key: /path/to/your/key.pem
```

## Features

- HTTPS-only configuration (HTTP redirects to HTTPS)
- Reverse proxy for Grafana at /grafana/
- WSGI configuration for Graphite at /graphite/
- WebSocket support for Grafana Live
- Self-signed SSL certificates by default

## License

MIT

## Author

Yinchuan Song (Red Hat)
