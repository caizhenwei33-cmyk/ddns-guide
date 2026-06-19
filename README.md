# Dynamic DNS (DDNS) Multi-Solution Tutorial / 动态 DNS (DDNS) 多方案教程

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> **Language:** English | [中文](#chinese-section)

---

<a name="chinese-section"></a>

# 动态 DNS (DDNS) 多方案教程

## 📋 目录 / Table of Contents

- [English](#english)
  - [1. Introduction](#1-introduction)
  - [2. How DDNS Works](#2-how-ddns-works)
  - [3. Solution Comparison](#3-solution-comparison)
  - [4. Cloudflare API DDNS](#4-cloudflare-api-ddns)
  - [5. Aliyun DNS DDNS](#5-aliyun-dns-ddns)
  - [6. Custom Script DDNS](#6-custom-script-ddns)
  - [7. Docker DDNS](#7-docker-ddns)
  - [8. Common Commands](#8-common-commands)
  - [9. FAQ](#9-faq)
- [中文](#chinese-section)
  - [1. 简介](#1-简介)
  - [2. DDNS 原理](#2-ddns-原理)
  - [3. 方案对比](#3-方案对比)
  - [4. Cloudflare API DDNS](#4-cloudflare-api-ddns-1)
  - [5. 阿里云 DNS DDNS](#5-阿里云-dns-ddns)
  - [6. 自定义脚本 DDNS](#6-自定义脚本-ddns)
  - [7. Docker DDNS](#7-docker-ddns-1)
  - [8. 常用命令](#8-常用命令)
  - [9. 常见问题](#9-常见问题)
- [Support / 支持](#-支持--support)

---

<a name="english"></a>

## 1. Introduction

**Dynamic DNS (DDNS)** is a service that automatically updates DNS records when your IP address changes. It is essential for:

- Hosting services (web, game, VPN) on a home connection with dynamic IP
- Remote access to home NAS, cameras, or smart home systems
- Maintaining stable access to self-hosted servers without static IP
- Bypassing ISP-assigned dynamic IP limitations

This tutorial covers **5 practical DDNS solutions** using major DNS providers, with step-by-step deployment guides for both Docker and native Linux environments.

---

## 2. How DDNS Works

### Basic Flow

```
[Home Router / Server] -- Dynamic IP --> [DDNS Client]
                                          |
                                          | (1) Detect current public IP
                                          | (2) Compare with last known IP
                                          | (3) If changed → call DNS API
                                          v
                              [DNS Provider API]
                                          |
                                          | Update A / AAAA record
                                          v
                              [yourdomain.com --> new IP]
```

### Key Components

| Component | Role |
|---|---|
| **DDNS Client** | Runs on your machine, checks IP changes |
| **Public IP Detection** | Uses services like `ifconfig.me`, `api.ipify.org` |
| **DNS API** | Provider's API to update records (Cloudflare, Aliyun, etc.) |
| **DNS Record** | A (IPv4) or AAAA (IPv6) record for your domain |
| **TTL** | Time-To-Live — how long DNS resolvers cache the record |

---

## 3. Solution Comparison

| Solution | Provider | Complexity | Cost | Reliability | Features |
|---|---|---|---|---|---|
| **Cloudflare API** | Cloudflare | ★★☆ | Free | ★★★★★ | Global CDN, proxy, API token |
| **Aliyun DNS** | Alibaba Cloud | ★★☆ | Free | ★★★★★ | Chinese mainland optimized |
| **Custom Script** | Any (generic) | ★★★ | Free | ★★★☆ | Maximum flexibility |
| **Docker DDNS** | Multi-provider | ★☆☆ | Free | ★★★★ | Easy deployment |
| **DDNS-GO** | Multi-provider | ★☆☆ | Free | ★★★★ | Web UI, 50+ providers |

### Provider Feature Matrix

| Feature | Cloudflare | Aliyun | Route53 (AWS) | Google DNS | DNSPod |
|---|---|---|---|---|---|
| Free Tier | ✅ | ✅ | ❌ ($0.50/zone) | ✅ | ✅ |
| API Token | ✅ | ✅ | ✅ | ✅ | ✅ |
| IPv6 Support | ✅ | ✅ | ✅ | ✅ | ✅ |
| Chinese Speed | ⚠️ Slow 🚀 Fast | 🚀 Fast | ⚠️ Slow | ❌ Blocked | 🚀 Fast |
| Wildcard Record | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 4. Cloudflare API DDNS

### 4.1 Prerequisites

```bash
# 1. Install curl and jq
apt update && apt install -y curl jq
# or
yum install -y curl jq

# 2. Get your Cloudflare API Token
#    Login to Cloudflare Dashboard → My Profile → API Tokens → Create Token
#    Permissions: Zone > DNS > Edit
#    Zone Resources: Include > Specific Zone > your-domain.com
```

### 4.2 Native Script

```bash
#!/bin/bash
# /usr/local/bin/cloudflare-ddns.sh

# Configuration
ZONE_NAME="yourdomain.com"          # Your domain
RECORD_NAME="home.yourdomain.com"   # Subdomain to update
API_TOKEN="YOUR_CLOUDFLARE_API_TOKEN"  # API Token
PROXY=false                         # Cloudflare proxy (CDN) on/off
TTL=120                             # TTL in seconds (120 = auto)

# Get current public IP
CURRENT_IP=$(curl -s https://api.ipify.org)
echo "Current public IP: $CURRENT_IP"

# Get Zone ID
ZONE_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones?name=$ZONE_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" | jq -r '.result[0].id')

if [ -z "$ZONE_ID" ] || [ "$ZONE_ID" = "null" ]; then
  echo "ERROR: Failed to get Zone ID"
  exit 1
fi
echo "Zone ID: $ZONE_ID"

# Get existing DNS record
RECORD_RESP=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?type=A&name=$RECORD_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json")

RECORD_ID=$(echo "$RECORD_RESP" | jq -r '.result[0].id')
RECORD_IP=$(echo "$RECORD_RESP" | jq -r '.result[0].content')

echo "Current DNS record IP: $RECORD_IP"

# Compare IPs
if [ "$CURRENT_IP" = "$RECORD_IP" ]; then
  echo "IP unchanged. No update needed."
  exit 0
fi

# Update DNS record
if [ -z "$RECORD_ID" ] || [ "$RECORD_ID" = "null" ]; then
  # Create new record
  echo "Creating new DNS record..."
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "Content-Type: application/json" \
    --data "{\"type\":\"A\",\"name\":\"$RECORD_NAME\",\"content\":\"$CURRENT_IP\",\"ttl\":$TTL,\"proxied\":$PROXY}" | jq .
else
  # Update existing record
  echo "Updating DNS record $RECORD_ID from $RECORD_IP to $CURRENT_IP..."
  curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "Content-Type: application/json" \
    --data "{\"type\":\"A\",\"name\":\"$RECORD_NAME\",\"content\":\"$CURRENT_IP\",\"ttl\":$TTL,\"proxied\":$PROXY}" | jq .
fi

echo "DDNS update completed!"
```

#### Install & Configure Cron

```bash
# Make script executable
chmod +x /usr/local/bin/cloudflare-ddns.sh

# Test run
/usr/local/bin/cloudflare-ddns.sh

# Add to crontab (run every 5 minutes)
crontab -e
*/5 * * * * /usr/local/bin/cloudflare-ddns.sh >> /var/log/ddns.log 2>&1

# Or use systemd timer
cat > /etc/systemd/system/cloudflare-ddns.service << 'EOF'
[Unit]
Description=Cloudflare DDNS updater
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cloudflare-ddns.sh
EOF

cat > /etc/systemd/system/cloudflare-ddns.timer << 'EOF'
[Unit]
Description=Run Cloudflare DDNS every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now cloudflare-ddns.timer
```

### 4.3 Docker Deployment (Cloudflare)

```bash
# Using oznu/cloudflare-ddns
docker run -d \
  --name cloudflare-ddns \
  --restart unless-stopped \
  -e API_KEY=YOUR_CLOUDFLARE_API_TOKEN \
  -e ZONE=yourdomain.com \
  -e SUBDOMAIN=home \
  -e PROXIED=false \
  oznu/cloudflare-ddns

# Using docker-compose
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  ddns:
    image: oznu/cloudflare-ddns
    container_name: cloudflare-ddns
    restart: unless-stopped
    environment:
      - API_KEY=YOUR_CLOUDFLARE_API_TOKEN
      - ZONE=yourdomain.com
      - SUBDOMAIN=home
      - PROXIED=false
      - TTL=120
EOF

docker-compose up -d
```

### 4.4 IPv6 Support (Cloudflare)

Modify the script for AAAA records:

```bash
# Change record type to AAAA
RECORD_TYPE="AAAA"

# Get IPv6 address
CURRENT_IP=$(curl -s https://api6.ipify.org)
# Or get from interface
CURRENT_IP=$(ip -6 addr show | grep 'global' | grep -v 'temporary' | awk '{print $2}' | cut -d/ -f1 | head -1)
```

---

## 5. Aliyun DNS DDNS

### 5.1 Prerequisites

```bash
# Install Aliyun CLI or use direct API
pip3 install aliyun-python-sdk-core aliyun-python-sdk-domain

# Get AccessKey from Aliyun RAM console
# https://ram.console.aliyun.com/
```

### 5.2 Native Script (Aliyun API)

```bash
#!/bin/bash
# /usr/local/bin/aliyun-ddns.sh

# Configuration
ACCESS_KEY_ID="YOUR_ACCESS_KEY_ID"
ACCESS_KEY_SECRET="YOUR_ACCESS_KEY_SECRET"
DOMAIN="yourdomain.com"
RR="home"           # Subdomain prefix (home.yourdomain.com)
TYPE="A"            # A or AAAA

# Get current public IP
if [ "$TYPE" = "AAAA" ]; then
  CURRENT_IP=$(curl -s https://api6.ipify.org)
else
  CURRENT_IP=$(curl -s https://api.ipify.org)
fi
echo "Current IP: $CURRENT_IP"

# Build API parameters
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
NONCE=$(date +%s)

# Query existing DNS record
function aliyun_request() {
  local action="$1"
  local params="$2"
  
  local query="AccessKeyId=$ACCESS_KEY_ID"
  query+="&Action=$action"
  query+="&Format=json"
  query+="&SignatureMethod=HMAC-SHA1"
  query+="&SignatureNonce=$NONCE"
  query+="&SignatureVersion=1.0"
  query+="&Timestamp=$TIMESTAMP"
  query+="&Version=2015-01-09"
  query+="&$params"
  
  # Calculate signature
  local string_to_sign="GET&%2F&$(echo -n "$query" | xxd -p | tr -d '\n' | xxd -r -p | jq -sRr @uri)"
  local signature=$(echo -n "$string_to_sign" | openssl dgst -sha1 -hmac "$ACCESS_KEY_SECRET&" -binary | base64)
  
  curl -s "https://dns.aliyuncs.com/?$query&Signature=$(echo -n "$signature" | jq -sRr @uri)"
}

# DescribeDomainRecords
RESP=$(aliyun_request "DescribeDomainRecords" "DomainName=$DOMAIN&RRKeyWord=$RR&Type=$TYPE")
CURRENT_VALUE=$(echo "$RESP" | jq -r ".DomainRecords.Record[] | select(.RR==\"$RR\") | .Value")
RECORD_ID=$(echo "$RESP" | jq -r ".DomainRecords.Record[] | select(.RR==\"$RR\") | .RecordId")

echo "Current DNS value: $CURRENT_VALUE (RecordId: $RECORD_ID)"

if [ "$CURRENT_IP" = "$CURRENT_VALUE" ]; then
  echo "IP unchanged, no update needed."
  exit 0
fi

# Update record
if [ -n "$RECORD_ID" ] && [ "$RECORD_ID" != "null" ]; then
  echo "Updating DNS record..."
  aliyun_request "UpdateDomainRecord" "RecordId=$RECORD_ID&RR=$RR&Type=$TYPE&Value=$CURRENT_IP" | jq .
else
  echo "Creating new DNS record..."
  aliyun_request "AddDomainRecord" "DomainName=$DOMAIN&RR=$RR&Type=$TYPE&Value=$CURRENT_IP" | jq .
fi

echo "DDNS update completed!"
```

### 5.3 Docker Deployment (Aliyun)

```bash
# Using jeessy/ddns-go (supports Aliyun + many others)
docker run -d \
  --name ddns-go \
  --restart unless-stopped \
  -p 9876:9876 \
  -v ~/ddns-go:/root \
  jeessy/ddns-go

# Then visit http://YOUR_IP:9876 to configure via Web UI
```

---

## 6. Custom Script DDNS

### 6.1 Generic Python DDNS

```python
#!/usr/bin/env python3
"""
Generic DDNS updater supporting any provider with REST API.
Supports Cloudflare, Aliyun, Google DNS, and custom endpoints.
"""
import json
import logging
import os
import sys
import time
from typing import Optional

import requests

# Configuration
CONFIG = {
    "provider": "cloudflare",  # cloudflare, aliyun, generic
    "domain": "yourdomain.com",
    "subdomain": "home",
    "record_type": "A",  # A or AAAA
    "ttl": 120,
    "check_interval": 300,  # seconds
    "ip_services": [
        "https://api.ipify.org",
        "https://ifconfig.me/ip",
        "https://icanhazip.com",
        "https://checkip.amazonaws.com"
    ],
    "providers": {
        "cloudflare": {
            "api_token": "YOUR_TOKEN",
            "zone_name": "yourdomain.com"
        },
        "aliyun": {
            "access_key_id": "YOUR_KEY",
            "access_key_secret": "YOUR_SECRET"
        }
    }
}

# Logging setup
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("/var/log/ddns.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)


def get_public_ip(services: list, record_type: str = "A") -> Optional[str]:
    """Try multiple IP detection services."""
    for service in services:
        try:
            if record_type == "AAAA":
                # Prefer IPv6 service
                ip_service = service.replace("api.ipify.org", "api6.ipify.org")
                resp = requests.get(ip_service, timeout=10)
            else:
                resp = requests.get(service, timeout=10)
            
            if resp.status_code == 200:
                ip = resp.text.strip()
                logger.debug(f"Got IP {ip} from {service}")
                return ip
        except Exception as e:
            logger.warning(f"Failed to get IP from {service}: {e}")
            continue
    return None


class CloudflareDDNS:
    def __init__(self, config: dict):
        self.api_token = config["api_token"]
        self.zone_name = config["zone_name"]
        self.base_url = "https://api.cloudflare.com/client/v4"
        self.headers = {
            "Authorization": f"Bearer {self.api_token}",
            "Content-Type": "application/json"
        }
    
    def get_zone_id(self) -> Optional[str]:
        resp = requests.get(
            f"{self.base_url}/zones?name={self.zone_name}",
            headers=self.headers
        )
        data = resp.json()
        if data["success"] and len(data["result"]) > 0:
            return data["result"][0]["id"]
        logger.error(f"Failed to get zone ID: {data}")
        return None
    
    def get_record(self, zone_id: str, name: str, record_type: str) -> tuple:
        resp = requests.get(
            f"{self.base_url}/zones/{zone_id}/dns_records"
            f"?type={record_type}&name={name}",
            headers=self.headers
        )
        data = resp.json()
        if data["success"] and len(data["result"]) > 0:
            record = data["result"][0]
            return record["id"], record["content"]
        return None, None
    
    def update(self, name: str, ip: str, record_type: str = "A", ttl: int = 120):
        zone_id = self.get_zone_id()
        if not zone_id:
            return False
        
        record_id, existing_ip = self.get_record(zone_id, name, record_type)
        
        if existing_ip == ip:
            logger.info(f"IP unchanged: {ip}")
            return True
        
        payload = {
            "type": record_type,
            "name": name,
            "content": ip,
            "ttl": ttl,
            "proxied": False
        }
        
        if record_id:
            logger.info(f"Updating {name} from {existing_ip} -> {ip}")
            url = f"{self.base_url}/zones/{zone_id}/dns_records/{record_id}"
            resp = requests.put(url, headers=self.headers, json=payload)
        else:
            logger.info(f"Creating new record {name} -> {ip}")
            url = f"{self.base_url}/zones/{zone_id}/dns_records"
            resp = requests.post(url, headers=self.headers, json=payload)
        
        result = resp.json()
        if result.get("success"):
            logger.info("DNS update successful!")
            return True
        else:
            logger.error(f"DNS update failed: {result}")
            return False


def main():
    config = CONFIG
    provider_name = config["provider"]
    subdomain = f"{config['subdomain']}.{config['domain']}"
    record_type = config["record_type"]
    ttl = config["ttl"]
    
    # Initialize provider
    if provider_name == "cloudflare":
        ddns = CloudflareDDNS(config["providers"]["cloudflare"])
    else:
        logger.error(f"Unsupported provider: {provider_name}")
        sys.exit(1)
    
    # Get current IP
    current_ip = get_public_ip(config["ip_services"], record_type)
    if not current_ip:
        logger.error("Failed to detect public IP")
        sys.exit(1)
    
    logger.info(f"Current IP: {current_ip}")
    
    # Update DNS
    success = ddns.update(subdomain, current_ip, record_type, ttl)
    
    if success:
        logger.info("DDNS check completed successfully")
    else:
        logger.error("DDNS update failed")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### 6.2 Install Python Script

```bash
# Install dependencies
pip3 install requests

# Save script
chmod +x /usr/local/bin/ddns-update.py

# Test run
/usr/local/bin/ddns-update.py

# Add to crontab
echo "*/5 * * * * /usr/local/bin/ddns-update.py >> /var/log/ddns.log 2>&1" | crontab -
```

---

## 7. Docker DDNS

### 7.1 Using ddns-go (Multi-Provider, Web UI)

```bash
# Run with Docker
docker run -d \
  --name ddns-go \
  --restart unless-stopped \
  -p 9876:9876 \
  -v ~/ddns-go:/root \
  jeessy/ddns-go

# Or pull specific version
docker run -d \
  --name ddns-go \
  --restart unless-stopped \
  -p 9876:9876 \
  -v ~/ddns-go:/root \
  jeessy/ddns-go:v5.7.1

# docker-compose
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  ddns-go:
    image: jeessy/ddns-go
    container_name: ddns-go
    restart: unless-stopped
    ports:
      - "9876:9876"
    volumes:
      - ./ddns-go:/root
EOF
```

**Supported Providers in ddns-go:**
- Aliyun DNS / 阿里云 DNS
- DNSPod (Tencent) / DNSPod (腾讯云)
- Cloudflare
- Huawei Cloud / 华为云
- AWS Route53
- Godaddy
- Namecheap
- Google Domains
- + 40+ more providers

### 7.2 Using oznu/cloudflare-ddns

```bash
docker run -d \
  --name cf-ddns \
  --restart unless-stopped \
  -e API_KEY=YOUR_TOKEN \
  -e ZONE=yourdomain.com \
  -e SUBDOMAIN=home \
  -e PROXIED=false \
  -e TTL=120 \
  oznu/cloudflare-ddns
```

### 7.3 Using custom Docker image with script

```dockerfile
# Dockerfile
FROM alpine:latest

RUN apk add --no-cache curl jq bash

COPY cloudflare-ddns.sh /usr/local/bin/ddns.sh
RUN chmod +x /usr/local/bin/ddns.sh

# Run every 5 minutes
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

CMD ["/entrypoint.sh"]
```

```bash
# entrypoint.sh
#!/bin/sh
while true; do
  /usr/local/bin/ddns.sh
  sleep 300
done
```

---

## 8. Common Commands

### IP Detection

```bash
# Public IPv4
curl -s https://api.ipify.org
curl -s https://ifconfig.me/ip
curl -s https://icanhazip.com
curl -s https://checkip.amazonaws.com
curl -s https://ipinfo.io/ip

# Public IPv6
curl -s https://api6.ipify.org

# Local IP
hostname -I
ip addr show | grep 'inet ' | awk '{print $2}'
ip -6 addr show | grep 'global' | awk '{print $2}'
```

### DNS Query

```bash
# Query current DNS record
dig A home.yourdomain.com +short
nslookup home.yourdomain.com
host home.yourdomain.com

# Check propagation
dig @1.1.1.1 home.yourdomain.com
dig @8.8.8.8 home.yourdomain.com
dig @208.67.222.222 home.yourdomain.com

# Check TTL
dig home.yourdomain.com | grep -E "^home.*A"

# Using Python
python3 -c "import socket; print(socket.gethostbyname('home.yourdomain.com'))"
```

### Log Management

```bash
# View DDNS logs
tail -f /var/log/ddns.log
journalctl -u cloudflare-ddns.service -f

# Check timer status
systemctl status cloudflare-ddns.timer
systemctl list-timers --all | grep ddns

# Crontab management
crontab -l
crontab -e
```

### Testing & Debugging

```bash
# Simulate IP change (local test)
curl -s -H "Content-Type: application/json" \
  -X POST "https://api.cloudflare.com/client/v4/zones/ZONE_ID/dns_records/RECORD_ID" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"type":"A","name":"test","content":"1.2.3.4","ttl":120}'

# Force update
/usr/local/bin/cloudflare-ddns.sh --force

# Dry run (if supported)
/usr/local/bin/ddns-update.py --dry-run
```

---

## 9. FAQ

### Q1: How often should DDNS update?
**A:** For most home connections, every 5 minutes is sufficient. ISPs rarely change IPs more than once per day. Too frequent updates may hit API rate limits.

### Q2: My IP changed but DNS still shows old IP?
**A:** Check: 1) DNS TTL — wait for propagation. 2) Script permissions. 3) API token validity. 4) Log output for error messages. 5) Firewall blocking outgoing API calls.

### Q3: Is Cloudflare proxy (orange cloud) recommended?
**A:** For HTTP services — yes (DDoS protection + CDN). For SSH/VPN/game servers — no (proxy breaks non-HTTP protocols).

### Q4: Can I use DDNS with a custom domain from Freenom/other free DNS?
**A:** Most free DNS providers support API-based updates. Check if they have a REST API. Cloudflare offers free DNS for any domain.

### Q5: How to handle dual-stack (IPv4 + IPv6)?
**A:** Create two separate cron jobs — one for A record (IPv4) and one for AAAA (IPv6). Use different IP detection services for each.

### Q6: What is the best free DDNS service?
| Service | Domain Options | Update Method | Reliability |
|---|---|---|---|
| **Cloudflare** | Your own domain | API | ★★★★★ |
| **DuckDNS** | *.duckdns.org | HTTP API | ★★★★☆ |
| **No-IP** | *.no-ip.org + custom | HTTP API / Client | ★★★☆☆ |
| **FreeDNS (Afraid)** | *.moons.ru + custom | API / Client | ★★★☆☆ |
| **Aliyun DNS** | Your own domain | API | ★★★★★ |

### Q7: DDNS stopped working after router restart?
**A:** Container/cron was on the router. Consider: 1) Move DDNS to a separate always-on device (Raspberry Pi, NAS). 2) Use router's built-in DDNS feature. 3) Set container restart policy to `always`.

### Q8: API rate limits?
| Provider | Limit | Recommended Interval |
|---|---|---|
| Cloudflare | 1200 req/5min | ≥ 1 minute |
| Aliyun | 100 req/sec | ≥ 5 minutes |
| Google DNS | 500 req/day | ≥ 1 hour |
| Route53 | 5 req/sec | ≥ 1 minute |

### Q9: How to secure API tokens?
**A:** 1) Use environment variables instead of hardcoding. 2) Restrict API token permissions to minimum (DNS:Edit only). 3) Set IP whitelisting on API tokens. 4) Use read-only token for querying, separate write token for updates. 5) Store tokens in a vault or secret manager.

### Q10: Can I update multiple subdomains?
**A:** Yes. Modify the script to loop through an array of records:

```bash
SUBDOMAINS=("home" "nas" "vpn" "blog")
for SUB in "${SUBDOMAINS[@]}"; do
  /usr/local/bin/cloudflare-ddns.sh "$SUB"
done
```

---

## 中文部分

## 1. 简介

**动态 DNS (DDNS)** 是一种在 IP 地址变化时自动更新 DNS 记录的服务。它对于以下场景至关重要：

- 在动态 IP 的家宽线路上托管服务（网站、游戏、VPN）
- 远程访问家庭 NAS、监控摄像头或智能家居系统
- 无静态 IP 时保持自建服务器的稳定访问
- 绕过 ISP 分配的动态 IP 限制

本教程涵盖 **5 种实用的 DDNS 方案**，使用主流 DNS 服务商，提供 Docker 和原生 Linux 两种部署方式的详细步骤。

## 2. DDNS 原理

### 基本流程

```
[家庭路由器/服务器] -- 动态 IP --> [DDNS 客户端]
                                   |
                                   | (1) 检测当前公网 IP
                                   | (2) 对比上次记录的 IP
                                   | (3) 如有变化 → 调用 DNS API
                                   v
                              [DNS 服务商 API]
                                   |
                                   | 更新 A / AAAA 记录
                                   v
                              [yourdomain.com --> 新 IP]
```

### 关键组件

| 组件 | 作用 |
|---|---|
| **DDNS 客户端** | 运行在本机，检查 IP 变化 |
| **公网 IP 检测** | 使用 `ifconfig.me`、`api.ipify.org` 等服务 |
| **DNS API** | 服务商的 API，用于更新记录 |
| **DNS 记录** | 域名的 A (IPv4) 或 AAAA (IPv6) 记录 |
| **TTL** | 生存时间 — DNS 解析器缓存该记录的时间 |

## 3. 方案对比

| 方案 | 服务商 | 复杂度 | 费用 | 可靠性 | 特点 |
|---|---|---|---|---|---|
| **Cloudflare API** | Cloudflare | ★★☆ | 免费 | ★★★★★ | 全球 CDN、代理、API Token |
| **阿里云 DNS** | 阿里云 | ★★☆ | 免费 | ★★★★★ | 中国大陆优化 |
| **自定义脚本** | 通用 | ★★★ | 免费 | ★★★☆ | 最大灵活性 |
| **Docker DDNS** | 多服务商 | ★☆☆ | 免费 | ★★★★ | 部署简单 |
| **DDNS-GO** | 多服务商 | ★☆☆ | 免费 | ★★★★ | Web 界面，50+ 服务商 |

### 服务商功能对比

| 功能 | Cloudflare | 阿里云 | Route53 (AWS) | Google DNS | DNSPod |
|---|---|---|---|---|---|
| 免费套餐 | ✅ | ✅ | ❌ ($0.50/zone) | ✅ | ✅ |
| API Token | ✅ | ✅ | ✅ | ✅ | ✅ |
| IPv6 支持 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 国内访问速度 | ⚠️ 慢 | 🚀 快 | ⚠️ 慢 | ❌ 被屏蔽 | 🚀 快 |
| 泛域名记录 | ✅ | ✅ | ✅ | ✅ | ✅ |

## 4. Cloudflare API DDNS

### 4.1 前置准备

```bash
# 1. 安装 curl 和 jq
apt update && apt install -y curl jq

# 2. 获取 Cloudflare API Token
#    登录 Cloudflare 控制台 → 我的资料 → API Tokens → 创建 Token
#    权限: Zone > DNS > Edit
#    区域资源: Include > Specific Zone > your-domain.com
```

### 4.2 原生脚本

完整脚本请参考英文部分。主要步骤：

1. 获取当前公网 IP（`api.ipify.org`）
2. 查询 Cloudflare Zone ID
3. 查询现有 DNS 记录
4. 比较 IP，有变化则调用 API 更新
5. 通过 crontab 或 systemd timer 定时执行

### 4.3 Docker 部署

```bash
# 使用 oznu/cloudflare-ddns
docker run -d \
  --name cloudflare-ddns \
  --restart unless-stopped \
  -e API_KEY=YOUR_CLOUDFLARE_API_TOKEN \
  -e ZONE=yourdomain.com \
  -e SUBDOMAIN=home \
  -e PROXIED=false \
  oznu/cloudflare-ddns
```

### 4.4 IPv6 支持

修改脚本中的 `RECORD_TYPE` 为 `AAAA`，使用 `api6.ipify.org` 获取 IPv6 地址。

## 5. 阿里云 DNS DDNS

### 5.1 前置准备

```bash
# 安装阿里云 SDK
pip3 install aliyun-python-sdk-core aliyun-python-sdk-domain

# 从 RAM 控制台获取 AccessKey
# https://ram.console.aliyun.com/
```

### 5.2 原生脚本

阿里云 API 签名机制比 Cloudflare 复杂，需要 HMAC-SHA1 签名。完整的 Bash 脚本已在英文部分提供，关键 API 调用：

- `DescribeDomainRecords` — 查询现有记录
- `UpdateDomainRecord` — 更新记录
- `AddDomainRecord` — 新建记录

### 5.3 Docker 部署

推荐使用 `jeessy/ddns-go`，支持阿里云和 50+ 服务商，通过 Web UI 配置。

## 6. 自定义脚本 DDNS

### 6.1 通用 Python DDNS

完整的 Python 脚本在英文部分提供，支持：
- 多 IP 检测服务（备选机制）
- Cloudflare API 集成
- 可扩展的 provider 架构
- 完善的日志记录
- 自动创建/更新记录

### 6.2 安装 Python 脚本

```bash
pip3 install requests
chmod +x /usr/local/bin/ddns-update.py
echo "*/5 * * * * /usr/local/bin/ddns-update.py >> /var/log/ddns.log 2>&1" | crontab -
```

## 7. Docker DDNS

### 7.1 ddns-go（多服务商 + Web UI）

```bash
docker run -d \
  --name ddns-go \
  --restart unless-stopped \
  -p 9876:9876 \
  -v ~/ddns-go:/root \
  jeessy/ddns-go
```

访问 `http://你的IP:9876` 通过 Web 界面配置。

### 7.2 oznu/cloudflare-ddns

轻量级 Cloudflare 专用 DDNS 容器，通过环境变量配置。

### 7.3 自定义 Docker 镜像

通过 Dockerfile 将脚本打包成容器，适合需要定制化配置的场景。

## 8. 常用命令

### IP 检测

```bash
# 公网 IPv4
curl -s https://api.ipify.org
curl -s https://ifconfig.me/ip
curl -s https://icanhazip.com

# 公网 IPv6
curl -s https://api6.ipify.org

# 本地 IP
hostname -I
ip addr show | grep 'inet ' | awk '{print $2}'
```

### DNS 查询

```bash
# 查询当前 DNS 记录
dig A home.yourdomain.com +short
nslookup home.yourdomain.com
host home.yourdomain.com

# 检查全球传播情况
dig @1.1.1.1 home.yourdomain.com
dig @8.8.8.8 home.yourdomain.com
dig @208.67.222.222 home.yourdomain.com
```

### 日志管理

```bash
tail -f /var/log/ddns.log
journalctl -u cloudflare-ddns.service -f
systemctl status cloudflare-ddns.timer
```

## 9. 常见问题

### Q1: DDNS 应该多久更新一次？
**A:** 对大多数家庭宽带，每 5 分钟足够。ISP 很少一天内多次更换 IP。过于频繁的更新可能触发 API 频率限制。

### Q2: IP 变了但 DNS 还是旧 IP？
**A:** 检查：1) DNS TTL — 等待传播。2) 脚本执行权限。3) API Token 有效性。4) 日志输出的错误信息。5) 防火墙是否拦截出站 API 请求。

### Q3: Cloudflare 代理（橙色云）推荐开启吗？
**A:** 对 HTTP 服务 — 是的（DDoS 保护 + CDN 加速）。对 SSH/VPN/游戏服务器 — 不要开启（代理会破坏非 HTTP 协议）。

### Q4: 能用 DDNS 配合免费域名吗？
**A:** 多数免费 DNS 服务商支持 API 更新。Cloudflare 为任何域名提供免费 DNS 托管。

### Q5: 如何处理双栈（IPv4 + IPv6）？
**A:** 创建两个独立的定时任务 — 一个更新 A 记录（IPv4），一个更新 AAAA 记录（IPv6），使用不同的 IP 检测服务。

### Q6: 最好的免费 DDNS 服务是什么？

| 服务 | 域名选项 | 更新方式 | 可靠性 |
|---|---|---|---|
| **Cloudflare** | 自有域名 | API | ★★★★★ |
| **DuckDNS** | *.duckdns.org | HTTP API | ★★★★☆ |
| **No-IP** | *.no-ip.org + 自定义 | HTTP API / 客户端 | ★★★☆☆ |
| **FreeDNS (Afraid)** | *.moons.ru + 自定义 | API / 客户端 | ★★★☆☆ |
| **阿里云 DNS** | 自有域名 | API | ★★★★★ |

### Q7: 路由器重启后 DDNS 不工作了？
**A:** DDNS 客户端也在路由器上。考虑：1) 将 DDNS 迁移到常开设备（树莓派、NAS）。2) 使用路由器的内置 DDNS 功能。3) 设置容器重启策略为 `always`。

### Q8: API 频率限制？

| 服务商 | 限制 | 推荐间隔 |
|---|---|---|
| Cloudflare | 1200 次/5分钟 | ≥ 1 分钟 |
| 阿里云 | 100 次/秒 | ≥ 5 分钟 |
| Google DNS | 500 次/天 | ≥ 1 小时 |
| Route53 | 5 次/秒 | ≥ 1 分钟 |

### Q9: 如何保护 API Token 安全？
**A:** 1) 使用环境变量而非硬编码。2) 限制 Token 权限为最小范围（仅 DNS:Edit）。3) 设置 IP 白名单。4) 查询和更新使用不同的 Token。5) 使用密钥管理服务存储。

### Q10: 能更新多个子域名吗？
**A:** 可以。脚本支持遍历子域名数组，批量更新。

---

## ☕ 支持 / Support

如果这个教程对你有帮助，欢迎请我喝杯咖啡：

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
