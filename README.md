markdown
# docker-compose-dpi-bypass

Forward proxy based on [dumbproxy](https://github.com/SenseUnit/dumbproxy), deployed via Docker Compose. Designed for networks with DPI (Deep Packet Inspection) that block or reset connections from containers.

## Problem

When working through a mobile hotspot (laptop → phone → internet), the ISP applies DPI and may:

- reset TLS connections from Docker containers
- spoof DNS responses
- block traffic based on TLS fingerprint

Classic forward proxies (Envoy, Squid) **do not work** in such networks: DPI recognizes their TLS fingerprint and resets the connection.

## Solution

`dumbproxy` uses the standard Go TLS stack, which **looks like a regular client** and does not raise suspicion with DPI. It runs in Docker as an HTTP/HTTPS forward proxy.

## Architecture
Client (curl/browser/container)
│
▼
dumbproxy :8080 (Docker)
│
▼
Host (laptop)
│
▼
Phone (hotspot)
│
▼
Internet

text

## Stack

- Docker
- Docker Compose v2
- dumbproxy v1.52+

## Requirements

- Linux (Ubuntu 22.04+)
- Docker 20.10+
- Docker Compose v2

## Getting Started

```bash
git clone https://github.com/OneOneExe/docker-compose-dpi-bypass.git
cd docker-compose-dpi-bypass
docker compose up -d
Check status:

bash
docker compose ps
docker compose logs dumbproxy
Stop:

bash
docker compose down
Authentication
The proxy is protected by Basic authentication. Default credentials:

text
Username: admin
Password: admin
How it works
Client connects to the proxy at http://localhost:8080.

Proxy responds with 407 Proxy Authentication Required.

Client (or you manually) enters username and password.

Proxy verifies them and allows traffic.

Without correct credentials, the proxy will not pass any request.

Threats
The admin:admin password is temporary. It is only for the first run and verification.

If you leave it as is:

Anyone who knows the proxy address can use it. If the proxy is reachable from the local network (or from the internet, if the port is exposed), outsiders can route their traffic through it.

The proxy can be used as an open relay. This will get your IP into blacklists, and your ISP may block access.

Logs will contain foreign requests. You won't be able to distinguish your actions from someone else's.

Port 8080 is published on all interfaces (0.0.0.0:8080). If you are on an untrusted network (public Wi-Fi, office), anyone can reach this port. With a weak password, they will get access.

How to change username and password
Open docker-compose.yml.

Find the line:

yaml
command: -bind-address :8080 -auth 'static://?username=admin&password=admin'
Replace admin and admin with your own values. For example:

yaml
command: -bind-address :8080 -auth 'static://?username=myuser&password=S3cureP@ssw0rd!'
Restart:

bash
docker compose down
docker compose up -d
Important:

The password is stored in plain text in docker-compose.yml. Do not push the file to a public repository with a real password.

If you change credentials, update them everywhere you use the proxy (browser, curl, environment variables).

Use a long password (12+ characters), preferably with digits and special characters.

If the proxy is needed only locally
You can bind the port to 127.0.0.1 so it is not reachable from outside:

yaml
ports:
  - "127.0.0.1:8080:8080"
Then only processes on the same host can connect. For local development — recommended.

Verification
1. From the host (HTTPS via CONNECT)
bash
curl -x http://admin:admin@localhost:8080 https://google.com -v
Expected:

text
> CONNECT google.com:443 HTTP/1.1
< HTTP/1.1 200 OK
* CONNECT tunnel established, response 200
< HTTP/2 301
2. From the host (plain HTTP)
bash
curl -x http://admin:admin@localhost:8080 http://httpforever.com -I
Expected: HTTP/1.1 200 OK.

3. From another container
bash
docker run --rm --network docker-compose-dpi-bypass_default \
  -e https_proxy=http://admin:admin@dumbproxy:8080 \
  curlimages/curl -s -o /dev/null -w "%{http_code}\n" https://google.com
Expected: 301.

Usage
Command line
bash
# HTTPS
curl -x http://admin:admin@localhost:8080 https://google.com

# HTTP
curl -x http://admin:admin@localhost:8080 http://httpforever.com

# git
git -c http.proxy=http://admin:admin@localhost:8080 clone https://github.com/...
Browser
Firefox: Settings → Network → Proxy → Manual → HTTP proxy localhost:8080 → check "Use for all protocols". The browser will prompt for username and password on the first request.

Chrome: launch with --proxy-server="http://localhost:8080". Credentials — via extension or system settings.

System (Linux)
bash
export http_proxy=http://admin:admin@localhost:8080
export https_proxy=http://admin:admin@localhost:8080
export HTTP_PROXY=http://admin:admin@localhost:8080
export HTTPS_PROXY=http://admin:admin@localhost:8080
export NO_PROXY=localhost,127.0.0.1
Another container (Docker Compose)
yaml
services:
  myapp:
    image: someapp:latest
    environment:
      - http_proxy=http://admin:admin@dumbproxy:8080
      - https_proxy=http://admin:admin@dumbproxy:8080
      - HTTP_PROXY=http://admin:admin@dumbproxy:8080
      - HTTPS_PROXY=http://admin:admin@dumbproxy:8080
      - NO_PROXY=localhost,127.0.0.1
    networks:
      - default
Important: containers must be on the same Docker network so that the name dumbproxy resolves.

Container User
docker-compose.yml specifies:

yaml
user: "1000:1000"
This means the process inside the container runs not as root, but as a user with UID 1000 and GID 1000 (a regular Linux user).

Why:

Security. If a vulnerability is found in the application, an attacker gets only the privileges of user 1000, not root. Root in a container is a potential root on the host in case of container escape.

Compatibility. UID/GID 1000 is the standard for the first user in most Linux distributions.

When this may cause issues:

If the image needs to write to system directories (/var/log, /etc) — permissions will be insufficient.

dumbproxy has no such need: it does not write to disk, logs go to stdout.

If you see permission denied errors in logs — remove the user: "1000:1000" line or replace it with user: "0:0" (root).

How to check which user the container runs as:

bash
docker compose exec dumbproxy id
If the command does not work (scratch image, no id), check this way:

bash
docker inspect dumbproxy --format '{{.Config.User}}'
Expected: 1000:1000.

Resource Limits
docker-compose.yml sets limits: half a CPU core and 128 MB of memory.

yaml
cpus: "0.5"
mem_limit: 128M
This works with a regular docker compose up launch.

If you migrate the project to Docker Swarm — these lines must be rewritten: Swarm does not understand cpus and mem_limit in this form. For Swarm, limits are set in a separate deploy block. If it comes to that — let me know, I'll help rewrite.

Project Structure
text
docker-compose-dpi-bypass/
├── docker-compose.yml
├── README.md
├── LICENSE
└── docs/
    ├── README.md
    ├── compose-ps.txt
    ├── curl-host-google.txt
    ├── curl-host-cloudflare.txt
    ├── curl-host-http.txt
    ├── curl-container-google.txt
    └── logs.txt
Proof of Work
All checks are collected in the docs/ folder:

compose-ps.txt — container status

curl-host-google.txt — check from host (Google, HTTPS via CONNECT)

curl-host-cloudflare.txt — check from host (Cloudflare)

curl-host-http.txt — check from host (plain HTTP)

curl-container-google.txt — explanation why the in-container check was not performed

logs.txt — container logs

More details — in docs/README.md.

Limitations
Linux only. On macOS/Windows, Docker Desktop uses a VM, and the localhost:8080 scheme works differently.

UDP is not proxied through an HTTP proxy. For UDP — use a VPN.

ICMP (ping) does not work through the proxy.

Applications must respect http_proxy/https_proxy environment variables. Some (Java, Electron) require separate configuration.

HTTPS traffic is not decrypted — dumbproxy works via a CONNECT tunnel, encryption stays between the client and the website.

Why Not Envoy / Squid
Envoy uses a TLS fingerprint that DPI recognizes and resets the connection. Verified: 503 UC without any explainable reason.

Squid — same issue, plus requires a patch for CONNECT.

dumbproxy — uses standard Go TLS, looks like a regular client.

License
MIT

References
dumbproxy on GitHub

dumbproxy in Snap Store
