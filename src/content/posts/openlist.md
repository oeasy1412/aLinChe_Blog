---
title: OpenList网盘
published: 2025-06-12
description: Nginx + DDNS-GO + OpenList + Cloudflare tunnel
tags: [net]
category: net
draft: false
---

## Nginx
[Nginx GitHub 仓库](https://github.com/nginx/nginx)
```sh
docker run -d \
  --name my-nginx \
  -p 80:80 \
  -p 443:443 \
  -v /home/username/OpenList/my-nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /home/username/OpenList/my-nginx/conf.d:/etc/nginx/conf.d:ro \
  -v /home/username/OpenList/my-nginx/html:/usr/share/nginx/html:ro \
  -v /home/username/OpenList/my-nginx/logs:/etc/nginx/logs/nginx \
  -v /home/username/OpenList/my-nginx/ssl:/etc/nginx/ssl:ro \
  --restart=always \
  nginx

docker logs my-nginx | grep -i error

# docker exec -it my-nginx /bin/bash # 如果需要进入容器交互式操作
docker exec my-nginx nginx -t
docker exec my-nginx nginx -s reload
```
### nginx.conf
```sh
# config
vim my-nginx/nginx.conf
```
```nginx
user nginx;
worker_processes auto;
error_log logs/error.log error; # notice;
pid /run/nginx.pid;
master_process on;
worker_rlimit_nofile 65535;
events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
}

http {
    include mime.types;
    include sites-enabled/*.conf;
    include conf.d/*.conf;
    default_type application/octet-stream;
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" "$http_x_forwarded_for"';
    access_log logs/access.log main;
    sendfile on; # 开启高效文件传输模式
    tcp_nopush on; # 防止网络阻塞
    keepalive_timeout 30;
    keepalive_requests 1000;
    reset_timedout_connection on;
    large_client_header_buffers 4 64k;
    client_max_body_size 0; # 设置文件上传大小限制,0为不限制
    # 开启 gzip 压缩
    gzip on;
    gzip_http_version 1.1;
    gzip_comp_level 3; # 压缩等级:1压缩比最小,处理速度快.9压缩比最大,比较消耗cpu资源
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;    
    gzip_vary on;
    # 静态文件缓存优化
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    # SSL 优化
    ssl_protocols TLSv1.2 TLSv1.3; # 推荐的安全协议
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    upstream openlist_backend {
        server 172.17.0.1:5244; # OpenList 默认端口
        keepalive 32;
    }

    server {
        listen 80;
        listen [::]:80;
        server_name openlist.alinche.dpdns.org;
        return 301 https://$host$request_uri; # 自动跳转到HTTPS
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name openlist.alinche.dpdns.org;
        charset utf-8;

        ssl_certificate /etc/nginx/ssl/certs/cloudflare_cert.pem;
        ssl_certificate_key /etc/nginx/ssl/private/cloudflare_key.key;

        location / {
            proxy_pass http://openlist_backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;

            # 缓冲区和超时（针对大文件传输优化）
            proxy_buffering on;
            proxy_buffer_size 128k;
            proxy_buffers 8 256k;
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        location /nginx_status {
            stub_status;
            add_header X-Nginx-Status active;
            access_log off;
            keepalive_timeout 0;
            allow 127.0.0.1;
            # deny all;
        }
    }

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        location / {
            try_files $uri $uri/ =404;
        }
    }

    # server {
    #     listen 443 default_server;
    #     listen [::]:443 default_server;
    #     server_name _;
    #     root /usr/share/nginx/html;
    #     index index.html;
    #     location / {
    #         try_files $uri $uri/ =404;
    #     }
    # }
}
```

```html
<!-- HTML -->
<!-- vim my-nginx/html/index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Welcome to nginx!</title>
    <style>
        html {
            color-scheme: light dark;
        }
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
</head>
<body>
    <h1>你好！nginx</h1>
    <p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
    <p>For online documentation and support please refer to
        <a href="https://www.bilibili.com/video/BV1TW1LYkE59/">bilibili.com</a>.<br />
    </p>
    <p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## DDNS-GO
[DDNS-GO GitHub 仓库](https://github.com/jeessy2/ddns-go)
```sh
docker run -d --name ddns-go --restart=always --net=host -v /opt/ddns-go:/root jeessy/ddns-go
# 访问 IP:9876 进行管理
```

## OpenList
[OpenList GitHub 仓库](https://github.com/OpenListTeam/OpenList)
[OpenList 官方文档 CN](https://doc.oplist.org.cn/)
```sh
# 使用 docker
sudo chown -R 1001:1001 /etc/openlist
docker run -d --restart=unless-stopped -v /etc/openlist:/opt/openlist/data -p 5244:5244 -e UMASK=022 --name="openlist" openlistteam/openlist:latest

# 如考虑使用二进制文件
sudo vim /etc/systemd/system/openlist.service # 访问 IP:5244 进行管理
[Unit]
Description=OpenList Service
After=network.target
[Service]
Type=simple
WorkingDirectory=/home/username/OpenList
ExecStart=/home/username/OpenList/openlist server
Restart=always
RestartSec=30
User=username
[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable openlist.service
sudo systemctl restart openlist.service
sudo systemctl status openlist.service

journalctl -u openlist.service -f
```

## Cloudflare tunnel
```sh
# https://one.dash.cloudflare.com/<id>/networks/connectors
sudo apt-get install cloudflared # 要用官网的命令添加 apt repo
# docker
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token [YOUR_TOKEN]

cloudflared tunnel login

sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared

cloudflared tunnel list
journalctl -a -u cloudflared -n 10

# 强制使用 HTTP2
sudo vim /etc/systemd/system/cloudflared.service
ExecStart=/usr/bin/cloudflared --no-autoupdate tunnel run --protocol http2 --token ...
```
