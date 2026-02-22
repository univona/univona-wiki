# Nginx + TLS 配置

本指南介绍如何配置 nginx 反向代理和 TLS 加密，为 Univona Admin Server 提供 HTTPS 访问。

## 安装 nginx

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install nginx

# 启用开机自启
sudo systemctl enable nginx
```

## 反向代理配置

### 基础配置（含 WebSocket 支持）

创建 `/etc/nginx/sites-available/univona`：

```nginx
upstream univona_backend {
    server 127.0.0.1:3001;

    # 如有多节点，可在此添加
    # server 127.0.0.1:3002;

    keepalive 32;
}

server {
    listen 80;
    server_name your-domain.com;

    # HTTP 重定向到 HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # ─── TLS 证书 ──────────────────────────────────────────
    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # ─── TLS 参数 ──────────────────────────────────────────
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # ─── 安全头 ────────────────────────────────────────────
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:;" always;

    # ─── 日志 ──────────────────────────────────────────────
    access_log /var/log/nginx/univona_access.log;
    error_log  /var/log/nginx/univona_error.log;

    # ─── 请求体大小限制 ────────────────────────────────────
    client_max_body_size 50m;  # 与媒体上传限制一致

    # ─── API 和管理面板反向代理 ────────────────────────────
    location / {
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;

        # 代理头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        # 缓冲
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    # ─── WebSocket 路由 ────────────────────────────────────
    location /ws {
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;

        # WebSocket 升级头
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 代理头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 超时（保持长连接）
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        # 禁用缓冲
        proxy_buffering off;
    }

    # ─── 媒体文件（可选：直接由 nginx 提供静态文件） ───────
    # 如果媒体文件目录可被 nginx 直接访问，可以绕过应用服务
    # location /media/ {
    #     alias /var/lib/univona/media/;
    #     expires 30d;
    #     add_header Cache-Control "public, immutable";
    # }

    # ─── 健康检查（无需认证） ──────────────────────────────
    location /healthz {
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        access_log off;  # 健康检查不记录日志
    }
}
```

### 启用站点

```bash
# 创建符号链接
sudo ln -s /etc/nginx/sites-available/univona /etc/nginx/sites-enabled/

# 删除默认站点（可选）
sudo rm -f /etc/nginx/sites-enabled/default

# 测试配置
sudo nginx -t

# 重载配置
sudo systemctl reload nginx
```

## Let's Encrypt 证书（certbot）

### 安装 certbot

```bash
sudo apt install certbot python3-certbot-nginx
```

### 获取证书

```bash
# 自动配置 nginx（推荐）
sudo certbot --nginx -d your-domain.com

# 或仅获取证书（手动配置 nginx）
sudo certbot certonly --webroot -w /var/www/html -d your-domain.com
```

### 自动续期

certbot 安装后会自动配置续期定时器。验证：

```bash
# 检查定时器状态
sudo systemctl status certbot.timer

# 手动测试续期（不会实际续期）
sudo certbot renew --dry-run
```

### 续期后重载 nginx

创建 `/etc/letsencrypt/renewal-hooks/post/reload-nginx.sh`：

```bash
#!/bin/bash
systemctl reload nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

## 自签名证书（开发/内网）

用于开发环境或内网部署。不建议在公开生产环境使用。

### 生成自签名证书

```bash
# 创建证书目录
sudo mkdir -p /etc/nginx/ssl

# 生成私钥和证书（有效期 365 天）
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/univona.key \
  -out /etc/nginx/ssl/univona.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=Univona/CN=univona.local"
```

### 使用自签名证书

在 nginx 配置中替换证书路径：

```nginx
ssl_certificate     /etc/nginx/ssl/univona.crt;
ssl_certificate_key /etc/nginx/ssl/univona.key;
```

### 生成带 SAN 的证书（推荐）

```bash
# 创建配置文件
cat > /tmp/univona-ssl.cnf << 'EOF'
[req]
default_bits = 2048
prompt = no
distinguished_name = dn
x509_extensions = v3_req

[dn]
C = CN
ST = Beijing
L = Beijing
O = Univona
CN = univona.local

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = univona.local
DNS.2 = *.univona.local
IP.1 = 127.0.0.1
IP.2 = 192.168.1.100
EOF

# 生成证书
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/univona.key \
  -out /etc/nginx/ssl/univona.crt \
  -config /tmp/univona-ssl.cnf
```

## 安全头说明

| 头 | 值 | 说明 |
|------|------|------|
| `Strict-Transport-Security` | `max-age=63072000` | 强制使用 HTTPS（2 年） |
| `X-Content-Type-Options` | `nosniff` | 禁止 MIME 类型嗅探 |
| `X-Frame-Options` | `SAMEORIGIN` | 防止点击劫持 |
| `X-XSS-Protection` | `1; mode=block` | 启用 XSS 过滤 |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | 控制 Referrer 泄露 |
| `Content-Security-Policy` | `default-src 'self'` | 内容安全策略 |

> Admin Server 本身也通过 `security_headers` 中间件添加了安全头。nginx 层的安全头提供额外保护。

## 完整配置参考

以下是整合了所有推荐配置的完整 nginx 配置文件：

```nginx
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    # 基础设置
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;  # 隐藏 nginx 版本

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml text/javascript;

    # 速率限制
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/s;

    include /etc/nginx/sites-enabled/*;
}
```

```nginx
# /etc/nginx/sites-available/univona

upstream univona_backend {
    server 127.0.0.1:3001;
    keepalive 32;
}

# HTTP -> HTTPS 重定向
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # TLS
    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;

    # 安全头
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 请求体大小
    client_max_body_size 50m;

    # API
    location /api/ {
        limit_req zone=api burst=50 nodelay;
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
    }

    # 认证端点（更严格的速率限制）
    location /api/v1/auth/ {
        limit_req zone=auth burst=10 nodelay;
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket
    location /ws {
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
    }

    # 健康检查
    location /healthz {
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        access_log off;
    }

    # 管理面板（静态文件）
    location / {
        proxy_pass http://univona_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 验证配置

```bash
# 测试 nginx 配置语法
sudo nginx -t

# 重载配置
sudo systemctl reload nginx

# 测试 HTTPS 连接
curl -v https://your-domain.com/healthz

# 检查 TLS 证书信息
openssl s_client -connect your-domain.com:443 -servername your-domain.com < /dev/null 2>/dev/null | openssl x509 -noout -text
```
