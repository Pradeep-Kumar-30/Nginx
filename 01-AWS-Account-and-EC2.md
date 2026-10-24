# 1. AWS account, EC2, and first Nginx install

This guide creates an AWS account, launches an Ubuntu EC2 instance, connects
from Windows PowerShell, and verifies the Nginx welcome page.

## 1. Create and secure an AWS account

1. Go to [aws.amazon.com](https://aws.amazon.com/) and choose **Create an AWS
   Account**. Follow the current signup flow using an email address you control,
   account name, contact and billing details, and requested verification.
2. Choose an account-support plan appropriate for your needs. Review the
   billing terms shown during signup; do not assume that an account or service
   will be free.
3. Sign in as the account root user and enable multi-factor authentication
   (MFA) in the account's security settings. Store recovery information
   securely.
4. Create a billing budget and alert in **Billing and Cost Management**. An
   alert notifies you; it does not stop resources or cap spending.
5. For regular administration, use an appropriately permissioned IAM
   Identity Center user or IAM identity rather than root credentials. Never
   create or use root access keys for this walkthrough.

AWS signup screens and available identity options can change. Follow the
current AWS console's account-security recommendations.

## 2. Launch an Ubuntu EC2 instance

In the AWS Console, open **EC2** and select **Launch instance**:

1. Enter a recognizable instance name, such as `nginx-web-01`.
2. Select **Ubuntu Server 24.04 LTS** (64-bit x86 or ARM, matching the chosen
   instance type).
3. Choose a small instance type suitable for a test. Check its current price
   and regional availability before launching.
4. Create a new key pair, choose the OpenSSH `.pem` format, download it once,
   and keep it private. AWS cannot provide the same private key again.
5. Create or select a security group with these inbound rules:
   - SSH, TCP 22, source **My IP** (your current public IP as a `/32`), not the
     whole internet.
   - HTTP, TCP 80, source `0.0.0.0/0` and `::/0` if you intend a public IPv6
     website.
   - HTTPS, TCP 443, source `0.0.0.0/0` and `::/0` if you intend a public
     HTTPS website.
6. Leave outbound access sufficient for package installation and updates.
   Choose a VPC/subnet with internet connectivity and a public IPv4 address if
   you need to connect directly from your computer.
7. Review storage and other settings, then launch the instance.

Do not open SSH to all addresses as a convenience. If your public IP changes,
update the SSH rule to your new IP.

### Optional: install Nginx automatically with EC2 user data

Instead of manually installing Nginx after connecting, paste this script into
**Advanced details → User data** before launching the Ubuntu instance. Use
either this bootstrap method or the manual installation below; do not expect
user data to rerun automatically on every reboot.

```bash
#!/bin/bash
set -euxo pipefail

apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
cat > /var/www/html/index.html <<'HTML'
<!doctype html>
<html><head><title>Nginx on EC2</title></head>
<body><h1>Nginx is running on EC2</h1></body></html>
HTML
systemctl enable --now nginx
```

User data runs as root on the instance. To inspect first-boot output after
connecting, use:

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

## 3. Connect from Windows PowerShell

Wait for the instance state to become **Running** and its status checks to
pass. Copy its public IPv4 address from the EC2 console. In **PowerShell on
Windows**, set the key path and connect:

```powershell
$key = "$env:USERPROFILE\Downloads\YOUR_KEY.pem"
$ip = "YOUR_PUBLIC_IP"
ssh -i $key ubuntu@$ip
```

If OpenSSH reports that the private key permissions are too open, restrict the
key file to your Windows user, then retry:

```powershell
icacls $key /inheritance:r
icacls $key /grant:r "$($env:USERNAME):(R)"
ssh -i $key ubuntu@$ip
```

At the first connection, check and accept the host fingerprint only if it is
the instance you intended to reach. The SSH session is now a shell on Ubuntu;
subsequent `bash` commands in this guide run there.

## 4. Update Ubuntu and install Nginx

If you did **not** use the optional user-data script:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

If user data installed Nginx already, check that service instead of repeating
the package installation:

```bash
sudo systemctl status nginx --no-pager
```

Check the configuration and installed version:

```bash
sudo nginx -t
nginx -v
```

From your Windows browser, visit `http://YOUR_PUBLIC_IP/`. You should see the
Nginx welcome page. If the page does not load, check the instance status,
security-group HTTP rule, public IP, and Nginx service status before changing
configuration.

## 5. Nginx paths and service commands

On Ubuntu, the usual paths are:

| Purpose | Path |
| --- | --- |
| Main configuration | `/etc/nginx/nginx.conf` |
| Per-site configurations | `/etc/nginx/sites-available/` |
| Enabled site links | `/etc/nginx/sites-enabled/` |
| Default web root | `/var/www/html/` |
| Access and error logs | `/var/log/nginx/` |

Common service and diagnostic commands:

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl reload nginx
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
sudo nginx -t
sudo tail -n 100 /var/log/nginx/access.log
sudo tail -n 100 /var/log/nginx/error.log
sudo journalctl -u nginx --no-pager -n 100
```

Always run `sudo nginx -t` after a configuration edit and before a reload.

## 6. Distribution alternatives from the PDF

The remaining guides use Ubuntu commands. If using a compatible
Amazon Linux/CentOS/RHEL-family system instead, the PDF's package examples are:

```bash
sudo yum install -y nginx
sudo systemctl enable --now nginx
```

The default web root and configuration include paths can differ by
distribution. On systems using `firewalld`, allow services at the host firewall
as well as in the EC2 security group:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Ubuntu EC2 commonly relies on the EC2 security group for inbound filtering;
check the actual host firewall configuration rather than assuming `firewalld`
is installed.
