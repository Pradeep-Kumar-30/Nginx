# 6. AWS scalable architecture: final deployment walkthrough

This final guide brings the earlier steps together into the AWS architecture
described on the last slide of `NGINX.pdf`: Nginx serves static content and
proxies dynamic requests, traffic is balanced across backend instances, TLS
secures public access, and DNS directs the custom domain to the service.

## 1. Architecture

```text
Visitors
   |
   | HTTPS (443); HTTP (80) redirects to HTTPS
   v
DNS A/AAAA record
   |
   v
Public Nginx EC2 proxy
   |                         \
   | static files              \ private HTTP application port
   v                             v
Web root                 Backend EC2 instances (private IPs)
                                  |
                                  +-- optional backup backend
```

For a learning deployment, one Nginx instance and multiple private backends are
enough to exercise reverse proxying and load balancing. A single public Nginx
instance is still a single point of failure; this layout alone is not
high-availability.

## 2. Create the EC2 instances and network rules

1. Complete [account setup and first EC2 launch](./01-AWS-Account-and-EC2.md).
   Select the same AWS region and VPC for the proxy and backends.
2. Create a security group for the public proxy:
   - TCP 22 from your current IP only.
   - TCP 80 and 443 from intended public clients.
3. Create a separate backend security group:
   - Permit the application's TCP port (for example, 3000) with the **source
     set to the proxy security group**.
   - Permit SSH only through a controlled administrative path; do not expose
     the application port to the public internet.
4. Launch two or more backend EC2 instances in the private network with the
   application installed and listening on the private interface/port.
5. For the simple public-proxy example, give the proxy a stable address using
   an Elastic IP or another suitable AWS routing design. Review current
   address-related charges and allocation behavior.
6. Test each private backend from the proxy:

   ```bash
   curl -i --max-time 5 http://10.0.2.11:3000/
   curl -i --max-time 5 http://10.0.3.12:3000/
   ```

   Replace those private IPs with the actual addresses. Do not proceed until
   the proxy can reach each healthy backend.

The private instances need a secure way to receive software updates (for
example, a suitable NAT or controlled egress design). Do not add broad inbound
rules to compensate for missing outbound connectivity.

## 3. Configure Nginx static content and upstreams

Install Nginx using the earlier guide. On Ubuntu, add the upstream to the
`http` block in `/etc/nginx/nginx.conf`:

```nginx
upstream app_backends {
    server 10.0.2.11:3000;
    server 10.0.3.12:3000;
    server 10.0.4.13:3000 backup;
}
```

Create the site's static root and page:

```bash
sudo install -d -o "$USER" -g www-data -m 2755 /var/www/example.com/html
printf '%s\n' '<h1>Nginx on AWS</h1>' > /var/www/example.com/html/index.html
sudo chmod 644 /var/www/example.com/html/index.html
```

Create `/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ @application;
    }

    location @application {
        proxy_pass http://app_backends;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

This example serves matching static files directly and proxies missing paths
to the application. Adapt the URI routing to the application's real routes;
do not unintentionally send static assets or private paths to the backend.

Enable and test the site:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/example.com
sudo nginx -t
sudo systemctl reload nginx
```

## 4. Point DNS and enable HTTPS

1. Point the domain's A record at the proxy's stable public IPv4 address.
   Configure an AAAA record only if IPv6 is actually enabled and routed end to
   end.
2. Verify DNS and public HTTP reachability.
3. Follow the [HTTPS and Certbot guide](./03-HTTPS-and-Certbot.md) to install
   the certificate and configure TLS for `example.com` and `www.example.com`.
4. Confirm `https://example.com/` works and test certificate renewal:

   ```bash
   sudo certbot renew --dry-run
   ```

Keep 80 available if required for HTTP-to-HTTPS redirection and certificate
validation, and keep 443 open for HTTPS.

## 5. Verify balancing and failover

Test through the Nginx endpoint multiple times:

```bash
for i in $(seq 1 10); do curl -sS https://example.com/health; done
```

For a meaningful load-balancing test, the application should return an
instance identifier at a health/test endpoint. Confirm requests reach the
expected backends and inspect Nginx and application logs. Test failure
procedures in a controlled environment; do not stop production backends just
to experiment.

Nginx's `backup` upstream server receives requests when primary servers are
unavailable according to upstream behavior. It does not provide full
application health checks, automatic instance replacement, or data replication.

## 6. Production high availability considerations

To remove the public Nginx host as a single point of failure, consider an AWS
load balancer in front of multiple Nginx instances, or use an appropriately
designed managed load-balancing/auto-scaling architecture. A production design
also needs health checks, deployment/rollback procedures, monitoring and
alerts, log retention, backups, patching, least-privilege IAM, and a tested
recovery plan. These are architecture extensions; the PDF's upstream examples
alone do not configure them.

## 7. Final verification checklist

- [ ] Billing alert is configured and unneeded resources are stopped or removed.
- [ ] Nginx configuration passes `sudo nginx -t`.
- [ ] Nginx service is enabled and active.
- [ ] Security groups expose only required ports and trusted sources.
- [ ] The custom domain resolves to the intended public endpoint.
- [ ] HTTPS works and `sudo certbot renew --dry-run` succeeds.
- [ ] Static content and application routes return the expected responses.
- [ ] Proxy access to each backend is private and verified.
- [ ] Logs, monitoring, backups, and failure handling are understood.
- [ ] The architecture's remaining single points of failure are documented.
