# cf-caddy

A custom Caddy build that includes Cloudflare plugins for DNS automation.

## Included Plugins

This image is built with the following modules:

* **[Cloudflare DNS](https://github.com/caddy-dns/cloudflare):** DNS-01 ACME challenge support.
* **[Cloudflare IP Source](https://github.com/WeidiDeng/caddy-cloudflare-ip):** Dynamic trusted proxy configuration for Cloudflare ranges.

## Usage

### Docker Compose

```yaml
services:
  caddy:
    image: ghcr.io/buildplan/cf-caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    environment:
      - CF_API_TOKEN=your_cloudflare_api_token
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

### Example Caddyfile

This configuration enables the DNS challenge for SSL and trusts Cloudflare's proxy IPs.

```Caddyfile
{
    # Automatically trust Cloudflare proxy IPs using the installed plugin
    servers {
        trusted_proxies cloudflare
    }
}

# Reusable snippet for Cloudflare DNS Challenge
(cloudflare_tls) {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
    }
}

# --- Grafana Block Example ---
stats.mydomain.test {
    # Add cloudflare 
    import cloudflare_tls
    
    # Create per app/service log file
    log {
        output file /var/log/caddy/grafana-access.log {
            mode 0640
            roll_size 5MiB
            roll_keep 3
            roll_keep_for 180h
        }
        format json
    }
    
    # Route traffic securely
    route {
        header {
            Strict-Transport-Security "max-age=31536000; includeSubDomains"
            X-Frame-Options "SAMEORIGIN"
            X-Content-Type-Options "nosniff"
            X-XSS-Protection "0"
            Referrer-Policy "strict-origin-when-cross-origin"
            Permissions-Policy "camera=(), microphone=(), geolocation=()"
        }

        reverse_proxy http://grafana:3000
    }
}
```

## Environment Variables

| Variable | Description |
| --- | --- |
| `CF_API_TOKEN` | Cloudflare API Token with `Zone:DNS:Edit` permission. |
