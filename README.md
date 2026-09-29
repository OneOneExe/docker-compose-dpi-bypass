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

`dumbproxy` uses the standard Go TLS stack, which **looks like a regular client** and does not raise suspicion with DPI. It runs in Docker and works as an HTTP/HTTPS forward proxy.

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
Verification
1. From the host
bash
curl -x http://localhost:8080 https://google.com -v
Expected:

text
> CONNECT google.com:443 HTTP/1.1
< HTTP/1.1 200 OK
* CONNECT tunnel established, response 200
< HTTP/2 301
2. From another container
bash
docker run --rm --network docker-compose-dpi-bypass_default \
  -e https_proxy=http://dumbproxy:8080 \
  curlimages/curl -s -o /dev/null -w "%{http_code}\n" https://google.com
Expected: 301

3. HTTP (unencrypted)
bash
curl -x http://localhost:8080 http://example.org -v
Usage
Command line
bash
# HTTPS
curl -x http://localhost:8080 https://google.com

# HTTP
curl -x http://localhost:8080 http://example.org

# git
git -c http.proxy=http://localhost:8080 clone https://github.com/...
Browser
Firefox: Settings → Network → Proxy → Manual → HTTP proxy localhost:8080 → check "Use for all protocols".

Chrome: launch with --proxy-server="http://localhost:8080".

System (Linux)
bash
export http_proxy=http://localhost:8080
export https_proxy=http://localhost:8080
export HTTP_PROXY=http://localhost:8080
export HTTPS_PROXY=http://localhost:8080
export NO_PROXY=localhost,127.0.0.1
Another container (Docker Compose)
yaml
services:
  myapp:
    image: someapp:latest
    environment:
      - http_proxy=http://dumbproxy:8080
      - https_proxy=http://dumbproxy:8080
      - HTTP_PROXY=http://dumbproxy:8080
      - HTTPS_PROXY=http://dumbproxy:8080
      - NO_PROXY=localhost,127.0.0.1
    networks:
      - default
Important: containers must be on the same Docker network so that the name dumbproxy resolves.

Project Structure
text
docker-compose-dpi-bypass/
├── docker-compose.yml
├── README.md
├── .gitignore
├── LICENSE
└── docs/
    ├── logs.txt
    ├── curl-host-google.txt
    ├── curl-host-cloudflare.txt
    ├── curl-host-http.txt
    ├── curl-container-google.txt
    └── compose-ps.txt
Proof of Work
HTTPS through dumbproxy (from host)
See docs/curl-host-google.txt

Key lines:

text
< HTTP/1.1 200 OK
* CONNECT tunnel established, response 200
< HTTP/2 301
< location: https://www.google.com/
HTTPS from another container
text
$ docker run --rm --network docker-compose-dpi-bypass_default \
    -e https_proxy=http://dumbproxy:8080 \
    curlimages/curl -s -o /dev/null -w "%{http_code}\n" https://google.com
301
dumbproxy logs
text
dumbproxy | INFO  dumbproxy v1.52.1 ... Proxy server started.
Limitations
Linux only. On macOS/Windows, Docker Desktop uses a VM, and the localhost:8080 scheme works differently.

UDP is not proxied through an HTTP proxy. For UDP — use a VPN.

ICMP (ping) does not work through the proxy.

Applications must respect the http_proxy/https_proxy environment variables. Some (Java, Electron) require separate configuration.

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
