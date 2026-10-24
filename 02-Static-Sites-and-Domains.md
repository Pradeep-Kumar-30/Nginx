# 2. Static websites, Nginx configuration, and custom domains

This guide creates a website, configures an Nginx server block, and optionally
adds additional sites and a custom domain.

## 1. Understand the configuration structure

The main file is `/etc/nginx/nginx.conf`. Ubuntu's package includes files from
`/etc/nginx/sites-enabled/` inside its `http` block. The main configuration
hierarchy is:

```nginx
# Global settings, for example worker_processes
events {
    # Connection-level settings, for example worker_connections
}

http {
    # HTTP-wide settings, MIME types, and logs

    server {
        # A virtual host/domain
        location / {
            # Rules for matching request URIs
        }
    }
}
```

- **Global block:** process-wide settings and logging.
- **`events`:** connection handling and worker connection limits.
- **`http`:** HTTP defaults, MIME types, logging, and server blocks.
- **`server`:** a site/domain and listening port.
- **`location`:** behavior for a request path or pattern, such as `/api/`.

## 2. Create a static site

On the EC2 Ubuntu shell, create a web root and a simple test page:

```bash
sudo install -d -o "$USER" -g www-data -m 2755 /var/www/example.com/html
printf '%s\n' '<!doctype html><html><head><title>Example</title></head><body><h1>It works</h1></body></html>' > /var/www/example.com/html/index.html
sudo chmod 644 /var/www/example.com/html/index.html
```

Create the site configuration:

```bash
sudo tee /etc/nginx/sites-available/example.com >/dev/null <<'NGINX'
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX
```

Enable the site, check syntax, and reload:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/example.com
sudo nginx -t
sudo systemctl reload nginx
```

If the symlink already exists, do not create a second one. Inspect it with
`ls -l /etc/nginx/sites-enabled/` and edit the existing file as needed.

Check the response locally on EC2 and then from your computer:

```bash
curl -I -H 'Host: example.com' http://127.0.0.1/
```

Browse to `http://YOUR_PUBLIC_IP/` for IP-based testing. A domain-based
`server_name` will match only after DNS points the domain at this server.

To copy a local site from Windows, use **PowerShell on Windows**, not the
Ubuntu shell:

```powershell
$key = "$env:USERPROFILE\Downloads\YOUR_KEY.pem"
ssh -i $key ubuntu@YOUR_PUBLIC_IP "mkdir -p /tmp/example-site"
scp -i $key -r "C:\path\to\your\site\*" ubuntu@YOUR_PUBLIC_IP:/tmp/example-site/
```

Then on EC2, install the files into the web root and check ownership/read
permissions:

```bash
sudo mkdir -p /var/www/example.com/html
sudo cp -a /tmp/example-site/. /var/www/example.com/html/
sudo find /var/www/example.com/html -type d -exec chmod 755 {} \;
sudo find /var/www/example.com/html -type f -exec chmod 644 {} \;
```

## 3. Host multiple websites

Create a separate web root and `server` block for each hostname. For example,
create `/var/www/second.example.com/html` and
`/etc/nginx/sites-available/second.example.com` with:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name second.example.com www.second.example.com;

    root /var/www/second.example.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable it and reload:

```bash
sudo ln -s /etc/nginx/sites-available/second.example.com /etc/nginx/sites-enabled/second.example.com
sudo nginx -t
sudo systemctl reload nginx
```

Nginx chooses a site based on the request's hostname and listening address.
Avoid duplicate `server_name` values unless you intentionally configure a
default or shared virtual host.

## 4. Point a custom domain at EC2

1. In your domain provider's DNS control panel, create an **A** record for
   `example.com` pointing to the instance's public IPv4 address. Add a separate
   record for `www` if you want `www.example.com`.
2. Use the same hostnames in the Nginx `server_name` directive.
3. Ensure the EC2 security group permits inbound TCP 80 and (for HTTPS) TCP
   443 from intended clients.
4. Wait for DNS changes to propagate, then check the records:

   ```bash
   # Run on Linux/macOS or a shell with dig installed
   dig +short example.com
   ```

An EC2 public IPv4 address can change after a stop/start. For a stable address,
consider an Elastic IP and review its current charges and allocation rules.

## 5. Custom 404 page and IP restrictions

Add an error page to a site's document root:

```bash
printf '%s\n' '<h1>Page not found</h1>' | sudo tee /var/www/example.com/html/404.html
```

Add the following directives inside the relevant `server` block:

```nginx
error_page 404 /404.html;
location = /404.html {
    internal;
}
```

To restrict a path to a trusted IP, add a matching location (replace the
example address). Keep the allow/deny policy deliberate to avoid blocking
legitimate traffic:

```nginx
location /private/ {
    allow 203.0.113.10;
    deny all;
}
```

After any edit:

```bash
sudo nginx -t
sudo systemctl reload nginx
```
