# traefik_container_deploy

An Ansible role that automates the deployment of Traefik with Docker Compose,  
including directory setup, templated configuration files, and Cloudflare DNS-01  
support for automated Let's Encrypt certificate provisioning.

## Requirements

- Ansible >= 2.14
- Docker (can be installed via `geerlingguy.docker`)
- Docker Compose v2
- Cloudflare account and API token for DNS-01 challenge

## Rootless Docker and Port Configuration

When running Docker in rootless mode, Traefik **cannot bind to ports below 1024** (e.g., 80 and 443) directly.  

This role automatically handles **port redirection** from the standard HTTP and HTTPS ports to higher, configurable ports using iptables.  
You can adjust the ports with these variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `container_traefik_http_port` | 8080 | HTTP port for Traefik in rootless mode |
| `container_traefik_https_port` | 8443 | HTTPS port for Traefik in rootless mode |

> ⚠️ The role sets up iptables rules to redirect traffic from ports 80 → `container_traefik_http_port`  
> and 443 → `container_traefik_https_port`. Make sure your system allows iptables modifications (the tasks require `become: true`).


## Role Variables

All variables are defined in `defaults/main.yml`. Key variables include:

### Container Configuration
| Variable | Default | Description |
|----------|---------|-------------|
| `container_traefik_name` | `traefik` | Name of the Traefik container |
| `container_traefik_dockeruser_uid` | `{{ tenant_uid }}` | UID for Docker socket access |
| `container_traefik_http_port` | `80` | HTTP port exposed by Traefik |
| `container_traefik_https_port` | `443` | HTTPS port exposed by Traefik |
| `container_traefik_domain` | `"traefik.example.com"` | Main domain for Traefik dashboard |
| `container_traefik_san_domains` | `["example.com","example.org"]` | SAN domains for certificates |

### Cloudflare / DNS-01 Challenge
| Variable | Default | Description |
|----------|---------|-------------|
| `container_traefik_cloudflare_mail` | `"your-cloudflare-email@example.com"` | Cloudflare account email |
| `container_traefik_cloudflare_token` | `"your-cloudflare-token"` | Cloudflare API token |

### Authentication
| Variable | Default | Description |
|----------|---------|-------------|
| `container_traefik_auth` | `"basic"` | Authentication method for dashboard (`basic` or `sso`) |
| `container_traefik_basicauth_user` | `"admin"` | Username for basic auth |
| `container_traefik_basicauth_password` | `"yourpassword"` | Password for basic auth |

### Static Routers, Services, and ServersTransports
| Variable | Default | Description |
|----------|---------|-------------|
| `container_traefik_routers` | See defaults/main.yml | Predefined static routers (Host rules, entryPoints, middlewares) |
| `container_traefik_services` | See defaults/main.yml | Predefined services (loadBalancer servers, passHostHeader) |
| `container_traefik_serversTransports` | See defaults/main.yml | Server transport settings (e.g., insecureSkipVerify) |

> **Note:** Customize these variables in your playbook or in a `vars` file for your environment.

## Dependencies

- `geerlingguy.docker` – ensures Docker is installed and ready

## Example Playbook

```yaml
- hosts: servers
  become: true
  vars:
    tenant_uid: 1000
  roles:
    - role: traefik_container_deploy
      vars:
        container_traefik_domain: "traefik.mycompany.com"
        container_traefik_san_domains:
          - "mycompany.com"
          - "myotherdomain.com"
        container_traefik_cloudflare_mail: "me@mycompany.com"
        container_traefik_cloudflare_token: "xxxxxx"
```

## Cloudflare DNS01 Setup for Wildcard Certificates

This role supports automatic TLS certificate generation using **Cloudflare** as the DNS provider via the DNS-01 challenge.
To use it, follow these steps:

1. **Create a Cloudflare API Token**  
   - Go to your Cloudflare dashboard → My Profile → API Tokens.  
   - Create a token with **Zone:DNS:Edit** permissions for the domain(s) you want to secure.

2. **Configure Role Variables**  
   Set the following in `defaults/main.yml` or your playbook:

   ```yaml
   container_traefik_cloudflare_mail: "[your-cloudflare-email]"
   container_traefik_cloudflare_token: "[your-cloudflare-token]"
   container_traefik_domain: "example.com"
   container_traefik_san_domains:
     - "example.com"
     - "example.org"

3. **Switching between Staging and Production**

   * In `traefik.yml`, you can select the Let's Encrypt server:

   ```yaml
   certificatesResolvers:
     cloudflare:
       acme:
         email: "{{ container_traefik_cloudflare_mail }}"
         storage: acme.json
         dnsChallenge:
           provider: cloudflare
           resolvers:
             - "1.1.1.1:53"
             - "1.0.0.1:53"
         caServer: https://acme-v02.api.letsencrypt.org/directory   # Production (default)
         # caServer: https://acme-staging-v02.api.letsencrypt.org/directory  # Staging
   ```

   * **Staging**: Use this for testing to avoid hitting Let's Encrypt rate limits.
   * **Production**: Uncomment the prod URL once everything is verified.

4. **Ensure Required Files Exist**

   * The role will automatically create necessary Traefik directories and configuration files.
   * It will also generate the `.env` file with your Cloudflare credentials.

5. **Traefik will handle certificate generation**

   * On deployment, Traefik will request wildcard certificates for your domains using the Cloudflare API token.
   * Certificates are stored in `acme.json` inside the Traefik data folder.

> ⚠️ Make sure your Cloudflare API token has sufficient permissions to create TXT records for the DNS-01 challenge.