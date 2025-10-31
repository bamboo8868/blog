---
title: caddy安装使用教程
date: 2025-10-31
categories:
 - go
tags:
 - go 
---
# 使用 Caddy 作为静态服务器的最大优势是零配置 HTTPS（绑定域名时自动生效）和极简的配置语法，非常适合快速部署静态网站。

## 1.ubuntu快速安装教程
```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```


## 2. 使用命令
```
caddy run  --config /etc/caddy/caddyFile 
caddy reload -config /etc/caddy/caddyFile
```


## 3. 静态文件配置
```
example.com {
    root *  /www/public
    file_server
}
```



## 4.反向代理配置
```
example.com {
    reverse_proxy http://127.0.0.1:3000
}
```

## 5.压缩
```
example.com {
    encode gzip br
}
```