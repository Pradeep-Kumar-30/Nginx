# 5. Caching, operations, troubleshooting, and interview topics

This guide covers the PDF's caching, worker connection, logging, timeout,
HTTP-status, and protocol topics, plus practical checks for an EC2 Nginx host.

## 1. Enable proxy caching

Caching can serve eligible upstream responses without asking the backend for
every request. Add a cache zone inside the `http` context of
`/etc/nginx/nginx.conf`:

```nginx
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=my_cache:10m
    inactive=60m
    max_size=1g;
```

In the relevant proxied site's `location` block:

```nginx
location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 60m;
    proxy_cache_key "$scheme$request_uri";
    proxy_pass http://backend_servers;
    proxy_set_header Host $host;
    add_header X-Cache-Status $upstream_cache_status always;
}
```

If required by the installed package, create the cache directory and assign
permissions to the Nginx worker account:

```bash
sudo install -d -o www-data -g www-data -m 750 /var/cache/nginx
sudo nginx -t
sudo systemctl reload nginx
```

Check the response header using `curl -I https://example.com/`; upstream cache
status can include `MISS`, `HIT`, `BYPASS`, or `EXPIRED`. Do not cache
personalized, authenticated, or sensitive responses without rules that
correctly isolate users and honor the application's cache directives. Cache
policy is application-specific; the PDF's 60-minute example is not universally
safe.

## 2. Worker processes and connection limits

The PDF illustrates four workers and 10,000 worker connections, or a
theoretical 40,000 worker connection slots:

```nginx
worker_processes 4;

events {
    worker_connections 10000;
}
```

Treat this as an illustration, not a default recommendation. `worker_processes
auto;` is a common starting point for current Nginx builds. Actual concurrent
capacity depends on file-descriptor limits, memory, workload, upstream
connections, and other system constraints; the number is not automatically the
number of supported client connections.

Inspect the current process and service limits before tuning:

```bash
nproc
ulimit -n
sudo systemctl show nginx --property=LimitNOFILE
```

Change limits only after measuring load and understanding the operating
system's service configuration.

## 3. Logs and useful diagnostics

Ubuntu package defaults:

- Access log: `/var/log/nginx/access.log`
- Error log: `/var/log/nginx/error.log`
- Main configuration: `/etc/nginx/nginx.conf`

Useful commands:

```bash
sudo nginx -t
sudo nginx -T
sudo systemctl status nginx --no-pager
sudo journalctl -u nginx --no-pager -n 100
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
curl -I http://127.0.0.1/
```

`nginx -T` prints the effective configuration; review its output before sharing
it because it may reveal internal hostnames or configuration details. Do not
share private keys or secrets from server files.

Common HTTP statuses:

| Status | General meaning |
| --- | --- |
| 200 | Request succeeded |
| 301 / 302 | Redirect |
| 400 | Bad request |
| 403 | Forbidden |
| 404 | Not found |
| 429 | Too many requests |
| 500 | Server-side error |
| 502 | Bad gateway; often an unavailable or invalid upstream response |
| 503 | Service unavailable |
| 504 | Gateway timeout; often an upstream response timed out |

These meanings are general HTTP behavior; diagnose the specific request using
the access/error logs and the backend's own logs.

## 4. Timeouts

The PDF has a timeouts topic but does not give directives or values. The
following are **illustrative 60-second proxy timeouts**, not a universal
production policy:

```nginx
location / {
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    proxy_pass http://backend_servers;
}
```

Set values according to expected connection setup and application response
times. A longer timeout can keep resources occupied; a shorter timeout can
break legitimate long-running requests. Validate after editing:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 5. Protocols and HTTP/2

Nginx serves HTTP and can terminate HTTPS/TLS. The PDF asks how to enable
HTTP/2 and shows `listen 443 ssl http2;`; exact syntax depends on the Nginx
version. Check the installed version with `nginx -v`, use the syntax supported
by that version, then run `sudo nginx -t`. Certbot may have created the HTTPS
listener already, so avoid adding a conflicting duplicate.

## 6. Troubleshooting checklist

1. Confirm EC2 is **Running** and status checks pass.
2. Confirm the URL resolves to the intended instance:

   ```bash
   # On EC2
   curl -I -H 'Host: example.com' http://127.0.0.1/
   ```

3. Check Nginx syntax and service health:

   ```bash
   sudo nginx -t
   sudo systemctl status nginx --no-pager
   ```

4. Read Nginx logs and system service logs.
5. For a 502/504, verify the backend process, address, port, routes, and
   security-group access from the proxy host.
6. For a domain or certificate issue, verify DNS answers and that TCP 80/443
   reach this instance.
7. Verify both the EC2 security group and any host firewall; opening one does
   not change the other.

## 7. Interview questions listed in the PDF

The PDF includes these prompts; this list records the topics rather than
providing unsupported answers:

- What is Nginx, and how does it differ from Apache?
- How does Nginx as a web server differ from Nginx as a reverse proxy?
- What is the default configuration file?
- How do you enable HTTP/2?
- How do you troubleshoot Nginx?
- What does `proxy_pass` do?
- How do you configure load balancing?
- How does Nginx caching work?
- How do you set up SSL/TLS?
- How do you configure a custom 404 page?
- How do you block specific IP addresses?

The PDF also contains a user-data script that installs **Apache/httpd**, not
Nginx. Do not use that script as the Nginx installation procedure.
