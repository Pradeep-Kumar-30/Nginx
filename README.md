# Nginx on AWS EC2

This guide set takes you from creating an AWS account to deploying Nginx on
EC2, adding websites and HTTPS, and configuring proxying, load balancing,
caching, and a scalable AWS architecture. It expands on the topics and examples
in [`NGINX.pdf`](./NGINX.pdf).

## Follow the guides in order

1. [AWS account, EC2, and first Nginx install](./01-AWS-Account-and-EC2.md)
2. [Static websites, Nginx configuration, and custom domains](./02-Static-Sites-and-Domains.md)
3. [HTTPS with Let's Encrypt and Certbot](./03-HTTPS-and-Certbot.md)
4. [Reverse proxy, load balancing, and high availability](./04-Proxy-Load-Balancing-and-HA.md)
5. [Caching, operations, troubleshooting, and interview topics](./05-Caching-Operations-and-Topics.md)
6. [AWS scalable architecture: final deployment walkthrough](./06-AWS-Scalable-Architecture.md)

## Before you start

- The walkthrough uses **Ubuntu Server 24.04 LTS** on EC2. Commands run on the
  Linux instance are in `bash` blocks; commands marked **PowerShell** run on
  your Windows computer.
- Replace placeholders such as `YOUR_PUBLIC_IP`, `example.com`, and
  `YOUR_KEY.pem` with your own values. Never paste private keys, passwords, or
  AWS access keys into these files or commit them to a repository.
- EC2, public IPv4 addresses, storage, data transfer, and related AWS services
  may incur charges. Review current AWS pricing, create a billing budget, and
  stop or terminate resources when finished. A budget alert is not a spending
  cap.
- AWS Console wording can change. Use the equivalent current options if a
  screen or label differs.

## PDF topics covered

The individual guides cover installation; the configuration hierarchy;
web-root and log paths; multiple sites and DNS; TLS/HTTPS; reverse proxying;
load-balancing methods; backup upstreams; connection limits; caching; timeouts;
HTTP status codes and protocols; troubleshooting; the EC2 Apache user-data
example (identified as Apache, not Nginx); and the final AWS scalable
architecture topic. Where the PDF names a topic without supplying values or
configuration, the guides call that out and label any added example.
