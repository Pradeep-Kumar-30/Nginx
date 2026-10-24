# 4. Reverse proxy, load balancing, and high availability

Nginx can serve as a reverse proxy in front of an application and distribute
requests among multiple backend servers. The backend service must already be
running and reachable from EC2.

## 1. Reverse proxy to one backend

For an application listening on `127.0.0.1:3000`, create a site configuration
such as `/etc/nginx/sites-available/app.example.com`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable and validate it:

```bash
sudo ln -s /etc/nginx/sites-available/app.example.com /etc/nginx/sites-enabled/app.example.com
sudo nginx -t
sudo systemctl reload nginx
curl -I http://127.0.0.1/
```

Replace the hostname and backend address with your deployment's values. Do not
expose an application's backend port publicly unless it is intentionally
designed for that access; for a local app, bind it to loopback.

For a TLS-enabled site, use the Certbot configuration in
[the HTTPS guide](./03-HTTPS-and-Certbot.md) and put the proxy location in the
HTTPS server block (or let Certbot modify the existing matching server block).

## 2. Load balance multiple backends

Use a single `upstream` group and proxy to its name. For Ubuntu, put the
`upstream` declaration inside the `http` context in
`/etc/nginx/nginx.conf` (not inside a `server` block), then use the server
block in the site file:

```nginx
# Inside http { ... } in /etc/nginx/nginx.conf
upstream backend_servers {
    server 10.0.2.11:3000;
    server 10.0.3.12:3000;
    server 10.0.4.13:3000;
}
```

```nginx
# In /etc/nginx/sites-available/app.example.com
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The example addresses are private-network placeholders. Replace them with
reachable backend private IPs and ports. In AWS, attach a backend security
group to the instances and permit the application port **from the Nginx
proxy's security group**, not from the whole internet. Then test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Nginx uses round-robin distribution by default. Other methods from the PDF:

```nginx
upstream backend_servers {
    least_conn;
    server 10.0.2.11:3000;
    server 10.0.3.12:3000;
}
```

- **Round-robin:** default distribution across available servers.
- **Least connections:** add `least_conn;` inside the `upstream` block.
- **IP hash:** add `ip_hash;` inside the block for client-IP-based affinity.
- **Weighted round-robin:** for example,
  `server 10.0.2.11:3000 weight=3;`.

Use one balancing method per upstream group and confirm that it fits the
application's session and health-check behavior.

## 3. Configure an upstream backup

The `backup` parameter marks a server to receive traffic when primary
upstream servers are unavailable:

```nginx
upstream backend {
    server 10.0.2.11:3000;
    server 10.0.3.12:3000;
    server 10.0.4.13:3000 backup;
}
```

The backup must still be provisioned, reachable, and able to serve the same
application. This setting alone is not comprehensive health monitoring or
automatic recovery orchestration.

## 4. Test from the proxy host

Check that a backend is reachable before debugging Nginx:

```bash
curl -i --max-time 5 http://10.0.2.11:3000/
sudo nginx -t
sudo tail -n 100 /var/log/nginx/error.log
```

Check the instance's AWS route tables, subnet connectivity, network ACLs, and
security groups if the backend connection times out.
