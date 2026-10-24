# Nginx

Nginx is a high-performance web server that can also act as a reverse proxy,
load balancer, and response cache. This guide collects the setup commands,
configuration examples, and AWS deployment topics from [`NGINX.pdf`](./NGINX.pdf).

> The PDF's hands-on examples target an AWS EC2 Linux instance. Run Linux
> commands on the EC2 host, not in Windows PowerShell. Replace example domains,
> IP addresses, and backend ports with your own values.

## Contents

- [Prerequisites](#prerequisites)
- [Install and start Nginx](#install-and-start-nginx)
- [Useful paths and service commands](#useful-paths-and-service-commands)
- [Configure a static website](#configure-a-static-website)
- [Nginx configuration structure](#nginx-configuration-structure)
- [Host multiple websites and use a custom domain](#host-multiple-websites-and-use-a-custom-domain)
- [HTTPS with Let's Encrypt](#https-with-lets-encrypt)
- [Reverse proxy](#reverse-proxy)
- [Load balancing and high availability](#load-balancing-and-high-availability)
- [Caching](#caching)
- [Connections, logs, timeouts, and troubleshooting](#connections-logs-timeouts-and-troubleshooting)
- [AWS architecture notes](#aws-architecture-notes)
- [Other topics in the PDF](#other-topics-in-the-pdf)

## Prerequisites

- An AWS account and an EC2 Linux instance (the PDF's practical exercise).
- SSH access and permission to install packages and edit `/etc/nginx`.
- For a public website, an EC2 security group and host firewall that allow the
  required traffic (normally TCP 80 for HTTP and TCP 443 for HTTPS).
- For a custom domain, a registered domain and access to its DNS settings.

## Install and start Nginx

Choose the command for your Linux distribution:

```bash
# Amazon Linux / CentOS / RHEL-family systems
sudo yum install -y nginx

# Ubuntu / Debian-family systems
sudo apt update
sudo apt install -y nginx
```

Start Nginx and enable it at boot:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

If the host uses `firewalld`, allow HTTP (and HTTPS when needed):

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Also allow inbound HTTP/HTTPS in the EC2 security group's inbound rules.

Open `http://<EC2-public-IP>/` in a browser to check the default site.

## Useful paths and service commands

| Purpose | Path or command |
| --- | --- |
| Main configuration | `/etc/nginx/nginx.conf` |
| Default web root (common package default) | `/usr/share/nginx/html` |
| Logs (common package default) | `/var/log/nginx/` |
| Check configuration syntax | `sudo nginx -t` |
| Start service | `sudo systemctl start nginx` |
| Stop service | `sudo systemctl stop nginx` |
| Reload configuration without a full restart | `sudo systemctl reload nginx` |
| Restart service | `sudo systemctl restart nginx` |
| Check service status | `sudo systemctl status nginx` |
| Follow the error log | `sudo tail -f /var/log/nginx/error.log` |
| Follow the access log | `sudo tail -f /var/log/nginx/access.log` |

After editing configuration, run `sudo nginx -t` first. Reload only if the
syntax test succeeds.

## Configure a static website

Put the site's HTML, CSS, and other static files in a web root, for example
`/var/www/example.com/html/`. Configure a server block in
`/etc/nginx/nginx.conf` or in a file included by it, such as
`/etc/nginx/conf.d/example.com.conf`:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Create the web root and a simple test page:

```bash
sudo mkdir -p /var/www/example.com/html
echo '<h1>It works</h1>' | sudo tee /var/www/example.com/html/index.html
```

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Ensure the Nginx worker can read the files and traverse their parent
directories. The worker account and default web-root layout can vary by
distribution.

## Nginx configuration structure

The main configuration file is `/etc/nginx/nginx.conf`. Its main sections are:

```nginx
# Global settings, such as worker_processes
events {
    # Connection-level settings, such as worker_connections
}

http {
    # HTTP-wide settings, MIME types, and logging

    server {
        # Settings for one domain or virtual host
        location / {
            # Behavior for a matching request URI
        }
    }
}
```

- **Global block:** process-wide settings and logging.
- **`events` block:** connection handling, including worker connection limits.
- **`http` block:** HTTP settings and one or more virtual hosts.
- **`server` block:** rules for a domain or IP and listening port.
- **`location` block:** rules for matching request paths, such as `/api/`.

Keep each `server` block inside `http`. A configuration file may contain
multiple `server` blocks.

## Host multiple websites and use a custom domain

Create a separate `server` block and web root for each site. For example, add
one block for `example-one.com` and another for `example-two.com`, each with
its own `server_name`, `root`, and `index`.

For a public custom domain:

1. Create an **A** record in the domain's DNS settings pointing to the EC2
   instance's public IP (an Elastic IP is preferable if the address must remain
   stable).
2. Set the matching hostname in the Nginx `server_name` directive, for example:

   ```nginx
   server_name www.example.com example.com;
   ```

3. Allow the needed ports in the EC2 security group and host firewall.
4. Wait for DNS to resolve, then test the hostname in a browser.

## HTTPS with Let's Encrypt

Install Certbot using the package manager appropriate to your platform:

```bash
# macOS (Homebrew)
brew install certbot

# Windows (Chocolatey, as shown in the PDF)
choco install certbot -y

# CentOS / RHEL-family Linux
sudo yum install -y epel-release
sudo yum install -y certbot python3-certbot-nginx

# Ubuntu / Debian-family Linux
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

The PDF shows Certbot's standalone method. It temporarily needs local port 80,
so stop Nginx while requesting the certificate:

```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d yourdomain.com
sudo systemctl start nginx
```

The certificate and key are written under
`/etc/letsencrypt/live/yourdomain.com/`. Add an HTTPS server block (replace the
domain and paths as appropriate):

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/yourdomain.com/html;
    index index.html;
}
```

Test and reload the configuration:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Allow TCP 443 through the host firewall and EC2 security group. Plan certificate
renewal and verify it works with `sudo certbot renew --dry-run`; a certificate
request made with `certonly --standalone` does not itself configure Nginx.

## Reverse proxy

Nginx can accept client requests and forward them to an application server. For
an app listening on `127.0.0.1:3000`:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The application must be running and reachable at the address in `proxy_pass`.
Test and reload after adding the server block.

## Load balancing and high availability

Define an upstream group and proxy requests to it:

```nginx
http {
    upstream backend_servers {
        server 127.0.0.1:3000;
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend_servers;
        }
    }
}
```

Nginx uses round-robin distribution by default. Other methods shown in the PDF:

```nginx
upstream backend_servers {
    least_conn;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
}
```

- **Round-robin:** distributes requests across servers by default.
- **Least connections:** add `least_conn;` inside the `upstream` block.
- **IP hash:** add `ip_hash;` inside the `upstream` block for client-IP-based
  affinity.
- **Weighted round-robin:** assign a weight, such as
  `server 127.0.0.1:3000 weight=3;`.

For a backup server used when primary servers are unavailable, mark it with
`backup`:

```nginx
upstream backend {
    server 192.168.1.1:3000;
    server 192.168.1.2:3000;
    server 192.168.1.3:3000 backup;
}
```

The backend addresses must be reachable from the Nginx host. Nginx's upstream
`backup` designation is not a replacement for monitoring or a complete
multi-region high-availability design.

## Caching

The PDF describes proxy caching as a way to serve stored upstream responses and
reduce backend work. Example:

```nginx
http {
    proxy_cache_path /var/cache/nginx
        levels=1:2
        keys_zone=my_cache:10m
        inactive=60m
        max_size=1g;

    upstream backend_server {
        server 127.0.0.1:3000;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_cache my_cache;
            proxy_cache_valid 200 60m;
            proxy_cache_key "$scheme$request_uri";
            proxy_pass http://backend_server;
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

Ensure the Nginx worker can write to the cache path. Do not cache personalized
or sensitive responses unless the cache key and cache rules have been designed
to prevent one user's response being served to another.

## Connections, logs, timeouts, and troubleshooting

The PDF gives an example of `worker_processes 4` and
`worker_connections 10000`, yielding a theoretical 40,000 connections. A
corresponding configuration shape is:

```nginx
worker_processes 4;

events {
    worker_connections 10000;
}
```

This is an illustrative tuning example, not a universal recommendation. Actual
capacity depends on available file descriptors, memory, workload, and whether a
connection uses one or multiple file descriptors. Check operating-system limits
before raising these values.

Nginx logs are commonly in `/var/log/nginx/`; inspect the access and error logs
when troubleshooting. Useful checks:

```bash
sudo nginx -t
sudo systemctl status nginx
sudo journalctl -u nginx --no-pager -n 100
sudo tail -n 100 /var/log/nginx/error.log
```

The PDF includes a **timeouts** topic but does not provide timeout directives or
values. Choose proxy timeout settings based on the application and its expected
request duration rather than copying an unspecified value.

## AWS architecture notes

The PDF's architecture outline uses Nginx on EC2 to serve static and dynamic
content, forward requests to backend services, distribute traffic across
instances, use Let's Encrypt/Certbot for TLS, and point a custom domain at the
server. Production deployments should also account for instance health,
security-group rules, stable DNS targets, certificate renewal, and monitoring.

The PDF also contains this **Apache** EC2 user-data example. It installs Apache
(`httpd`), not Nginx; it is included for reference and is not required for the
Nginx setup above:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<html><h1>This is Website 1</h1></html>" > /var/www/html/index.html
```

## Other topics in the PDF

The source presentation also mentions HTTP status codes, protocols, and
interview questions. Its slides do not provide specific status-code guidance,
protocol configuration, timeout values, or answers. Interview prompts include:

- What is Nginx, and how does it differ from Apache?
- How does Nginx as a web server differ from Nginx as a reverse proxy?
- What is the default configuration file?
- How do you enable HTTP/2?
- How do you troubleshoot Nginx?
- What does `proxy_pass` do?
- How do you configure load balancing, caching, and SSL/TLS?
- How do you configure a custom 404 page or block specific IP addresses?

The PDF's TLS example uses `listen 443 ssl http2;`. HTTP/2 configuration
requirements can depend on the Nginx version and TLS setup; verify the
configuration with `sudo nginx -t` and consult the documentation for the
installed version.
