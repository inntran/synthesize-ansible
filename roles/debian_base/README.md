# debian_base

Base Debian system validation and package installation for the Synthesize project.

## Requirements

- Debian 13 (Trixie)

## Role Variables

See `defaults/main.yml` for all available variables:

- `base_build_packages`: List of build tools needed for compiling software
- `base_python_packages`: List of Python packages needed for Graphite ecosystem

## Dependencies

None

## Example Playbook

```yaml
- hosts: servers
  become: true
  roles:
    - debian_base
```

## License

MIT

## Author

Yinchuan Song (Red Hat)
