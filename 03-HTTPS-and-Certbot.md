# 3. HTTPS with Let's Encrypt and Certbot

HTTPS encrypts browser-to-server traffic using TLS. The PDF demonstrates
Let's Encrypt certificates and Certbot. Use a domain that already resolves to
the EC2 instance and keep inbound TCP 80 and 443 enabled in its security group.

## 1. Verify DNS and HTTP first

Before requesting a public certificate:

- Create DNS A records for the exact names to be included in the certificate,
  such as `example.com` and `www.example.com`.
- Confirm those names resolve to the public address of this Nginx instance.
- Confirm `http://example.com/` reaches the site from outside AWS.

Certificate authorities must be able to validate control of the requested
domain. A bare EC2 IP address cannot receive a normal public Let's Encrypt
certificate.

## 2. Install Certbot on Ubuntu

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

The Nginx plugin can obtain a certificate and configure the matching Nginx
site:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Follow the prompts, provide a valid email address, accept the terms, and choose
whether to redirect HTTP requests to HTTPS when prompted. Certbot updates the
matching Nginx configuration.

## 3. Verify HTTPS and renewal

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo certbot certificates
sudo systemctl status certbot.timer --no-pager
sudo certbot renew --dry-run
```

Open `https://example.com/` in a browser. Keep TCP 443 open in the EC2 security
group. Renewal is normally scheduled by the package's timer; investigate timer
or dry-run failures rather than assuming certificates renew successfully.

## 4. Certificate-only (standalone) method from the PDF

The PDF also shows `certbot certonly --standalone`. Standalone mode binds port
80 itself, so stop Nginx before issuance and start it again afterward:

```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d example.com
sudo systemctl start nginx
```

This method obtains files but does **not** configure Nginx. The certificate
files are typically stored under
`/etc/letsencrypt/live/example.com/`. Add an HTTPS server block using the
correct paths:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/example.com/html;
    index index.html;
}
```

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Use one issuance method at a time and avoid accidentally replacing a working
site configuration. Keep private key files readable only by the required
system processes.

## 5. Other installation commands listed in the PDF

These are the PDF's platform examples; the main walkthrough uses Ubuntu:

```bash
# CentOS / RHEL-family Linux (package availability varies by release)
sudo yum install -y epel-release
sudo yum install -y certbot python3-certbot-nginx

# macOS with Homebrew
brew install certbot

# Windows with Chocolatey
choco install certbot -y
```

For current package support and platform-specific installation instructions,
check Certbot's official documentation. Installing Certbot on a Windows
workstation does not configure the Certbot instance or Nginx on EC2.

## 6. TLS and HTTP/2

The PDF shows `listen 443 ssl http2;` with TLS 1.2 and 1.3. HTTP/2 syntax and
defaults depend on the installed Nginx version. Certbot may manage the TLS
server block; check the actual configuration and validate it with
`sudo nginx -t` rather than blindly adding duplicate `listen` directives.
