# Ansible Role: oauth2-proxy

[![Lint](https://github.com/zekker6/ansible-role-oauth2-proxy/actions/workflows/lint.yml/badge.svg)](https://github.com/zekker6/ansible-role-oauth2-proxy/actions/workflows/lint.yml)
[![Test](https://github.com/zekker6/ansible-role-oauth2-proxy/actions/workflows/test.yml/badge.svg)](https://github.com/zekker6/ansible-role-oauth2-proxy/actions/workflows/test.yml)

Installs and manages [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/) on Linux hosts. The role downloads the official release binary (with checksum verification), renders a configuration file, manages a systemd service, and optionally provisions static basic-auth users via an htpasswd file.

## Requirements

- Ansible (`ansible-core`) >= 2.15
- A systemd-based target host

The role uses only `ansible.builtin` modules - no extra collections, and nothing is installed on the target for htpasswd. Passwords for `oauth2_proxy_htpasswd_users` must be **pre-hashed** (bcrypt or SHA, the schemes oauth2-proxy supports); the role never handles plaintext. Generate an entry with:

```bash
htpasswd -nbB alice 's3cret'   # -> alice:$2y$05$...  (use the part after the colon)
```

## Role Variables

All variables are defined in [`defaults/main.yml`](defaults/main.yml).

### Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `oauth2_proxy_version` | `"7.15.2"` | oauth2-proxy release to install (no leading `v`). |
| `oauth2_proxy_user` / `oauth2_proxy_group` | `oauth2-proxy` | System account that runs the service. |
| `oauth2_proxy_install_dir` | `/opt/oauth2-proxy` | Where versioned binaries are extracted. |
| `oauth2_proxy_binary_path` | `/usr/local/bin/oauth2-proxy` | Symlink to the active binary. |
| `oauth2_proxy_config_dir` | `/etc/oauth2-proxy` | Configuration directory. |
| `oauth2_proxy_config_file` | `{{ oauth2_proxy_config_dir }}/oauth2-proxy.cfg` | Rendered config file. |

### Core configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `oauth2_proxy_http_address` | `"127.0.0.1:4180"` | Listen address. |
| `oauth2_proxy_https_address` | `""` | HTTPS listen address (optional). |
| `oauth2_proxy_upstreams` | `[]` | List of upstream URLs to proxy. |
| `oauth2_proxy_provider` | `""` | OAuth/OIDC provider (required unless `oauth2_proxy_validate_config: false`). |
| `oauth2_proxy_client_id` | `""` | Provider client ID. |
| `oauth2_proxy_client_secret` | `""` | Provider client secret. |
| `oauth2_proxy_cookie_secret` | `""` | Cookie seed (16, 24 or 32 bytes, raw or base64). Required (validated when `oauth2_proxy_validate_config` is true). Generate with `openssl rand -base64 32 \| tr -- '+/' '-_'`. |
| `oauth2_proxy_email_domains` | `[]` | Allowed email domains. Empty forces an explicit access decision (validated when `oauth2_proxy_validate_config` is true): set specific domains, use `["*"]` to allow any authenticated email, or set `authenticated_emails_file` via `oauth2_proxy_extra_options`. |
| `oauth2_proxy_cookie_secure` | `true` | Send cookies over HTTPS only. |
| `oauth2_proxy_reverse_proxy` | `false` | Trust `X-Forwarded-*` headers. |
| `oauth2_proxy_redirect_url` | `""` | OAuth callback URL (optional). |
| `oauth2_proxy_extra_options` | `{}` | Passthrough mapping for any other config-file option not listed above. |
| `oauth2_proxy_validate_config` | `true` | Assert required variables before configuring. |

Any option not exposed as a typed variable can be passed through `oauth2_proxy_extra_options`. Keys are config-file option names; values may be strings, booleans, integers or lists, and are rendered with the correct type:

```yaml
oauth2_proxy_extra_options:
  skip_provider_button: true
  whitelist_domains:
    - ".example.com"
  scope: "openid email profile"
```

### htpasswd (static basic-auth users)

| Variable | Default | Description |
|----------|---------|-------------|
| `oauth2_proxy_htpasswd_users` | `[]` | List of `{username, hashed_password}` mappings; `hashed_password` is a pre-computed bcrypt/SHA hash. When non-empty the htpasswd file is generated and wired into the config. |
| `oauth2_proxy_htpasswd_file` | `{{ oauth2_proxy_config_dir }}/htpasswd` | Path to the generated htpasswd file. |
| `oauth2_proxy_display_htpasswd_form` | `true` | Show the basic-auth login form. |

## Example Playbooks

### OIDC provider with an upstream

```yaml
- hosts: proxies
  become: true
  roles:
    - role: zekker6.oauth2_proxy
      vars:
        oauth2_proxy_provider: "oidc"
        oauth2_proxy_client_id: "{{ vault_oauth2_client_id }}"
        oauth2_proxy_client_secret: "{{ vault_oauth2_client_secret }}"
        oauth2_proxy_cookie_secret: "{{ vault_oauth2_cookie_secret }}"
        oauth2_proxy_redirect_url: "https://app.example.com/oauth2/callback"
        oauth2_proxy_email_domains:
          - "example.com"
        oauth2_proxy_upstreams:
          - "http://127.0.0.1:8080/"
        oauth2_proxy_reverse_proxy: true
        oauth2_proxy_extra_options:
          oidc_issuer_url: "https://id.example.com/"
```

### htpasswd-based basic auth

```yaml
- hosts: proxies
  become: true
  roles:
    - role: zekker6.oauth2_proxy
      vars:
        oauth2_proxy_provider: "google"
        oauth2_proxy_client_id: "{{ vault_oauth2_client_id }}"
        oauth2_proxy_client_secret: "{{ vault_oauth2_client_secret }}"
        oauth2_proxy_cookie_secret: "{{ vault_oauth2_cookie_secret }}"
        oauth2_proxy_upstreams:
          - "http://127.0.0.1:8080/"
        # Pre-hashed via `htpasswd -nbB <user> <password>` (store the hash in vault)
        oauth2_proxy_htpasswd_users:
          - username: alice
            hashed_password: "{{ vault_alice_htpasswd_hash }}"
          - username: bob
            hashed_password: "{{ vault_bob_htpasswd_hash }}"
```

## Development & Testing

Tooling is wired through [mise](https://mise.jdx.dev/) and [Task](https://taskfile.dev/):

```fish
mise install            # provision python, task, uv and the project venv
task deps               # install python deps + ansible collections
task lint               # yamllint + ansible-lint
task test               # full molecule sequence (default distro: debian13)

# test another distro
set -x MOLECULE_DISTRO ubuntu2404; task test
```

Molecule uses the Docker driver with [geerlingguy systemd-enabled images](https://hub.docker.com/u/geerlingguy). CI runs the same `task lint` / `task test` targets against Debian, Ubuntu, and Rocky Linux.

## License

MIT

## Author

Zakhar Bessarab
