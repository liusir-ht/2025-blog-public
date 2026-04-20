## 问题背景
  国内用户访问网站时，需要经过两层代理进行转发，前端页面访问后端都需要配置跨域配置，导致国内用户的请求经过了两层跨域，返回CROS错误。

## 环境说明
| 组件名称 | 组件版本 | 组件作用 |
| -- | -- | -- |
| Nginx | 1.20.1 | 接受用户请求，并进行路由转发 |
### 访问流程图

![](/blogs/resolve-cros/fb6c14a5a2bba5e2.png)


## 解决思路
  核心在于国内用户正常访问会返回跨域问题，主要是因为后端返回了重复的CROS的跨域请求头，导致无法正常处理跨域请求。**aws-bj**的Nginx不能动，所以主要还是解决**tx-ap**的Nginx的请求头返回。
1. 区分国内/国外用户的Server段配置
2. 国内用户访问代理时，tx-ap不配置CROS跨域配置，只保留基本转发和超时配置
3. 国外用户访问时，tx-ap配置CROS跨域配置，进行资源的转发访问。

## 代码实现
  首先在**tx-ap**地域配置一个单独的server块，进行请求路由区分，aws-bj、aws-ap在执行反向代理的时候，` proxy_set_header Host backend-ap-internal.ilivedata.com;`去替换代理的header头部信息，指定转入到tx-ap的国内server的路由部分，最终实现效果。
### aws-bj
```
server {
    listen 80;
    listen [::]:80;
    server_name video-translation-cn.ilivedata.com;
    server_name video-translation.ilivedata.com;
    return 301 https://$host$request_uri;
}

# HTTPS 代理服务器
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name backend-cn.ilivedata.com;
    server_name backend.ilivedata.com;
    server_name backend-route.ilivedata.com;
    server_name backend-ap-internal.ilivedata.com;


    # SSL 优化配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_ecdh_curve secp384r1;

    # HSTS 安全头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 代理配置
    location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-Device-ID,X-App-Source';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-App-Source';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
        proxy_pass https://backend/;  # 注意这里也使用https

        # 标准代理头设置
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # SSL 代理额外头
        proxy_ssl_server_name on;
        proxy_ssl_name $proxy_host;

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }
    location /video/dubbing/generateAiRewriteStream {

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-Device-ID,X-App-Source';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-Device-ID,X-App-Source';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
        add_header 'X-Accel-Buffering' 'no';

        #proxy_set_header Host $host;
        proxy_set_header Host video-translation-ap-internal.ilivedata.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 180s;
        proxy_send_timeout 60s;
        proxy_read_timeout 120s;
        proxy_pass      https://backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        chunked_transfer_encoding on;
        gzip off;
        proxy_cache off;
        add_header X-Accel-Buffering no ;
    }
    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}

upstream backend {
     server backend-ap-aws.ilivedata.com:443;
 }
```
### aws-ap
```
server {
    listen 443 ssl http2;
    server_name backend.ilivedata.com;
    server_name backend-ap-aws.ilivedata.com;
    server_name backend-cn.ilivedata.com;
    server_name backend-route.ilivedata.com;
    server_name backend-ap-internal.ilivedata.com;
    # SSL 优化配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_ecdh_curve secp384r1;

    # HSTS 安全头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 代理配置
    location / {
        proxy_pass https://backend/;

        # 标准代理头设置
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # SSL 代理额外头
        proxy_ssl_server_name on;
        proxy_ssl_name $proxy_host;

        # 超时设置
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }
    location /video/dubbing/generateAiRewriteStream {


        proxy_set_header Host backend-ap-internal.ilivedata.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 180s;
        proxy_send_timeout 60s;
        proxy_read_timeout 120s;
        proxy_pass      https://backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        chunked_transfer_encoding on;
        gzip off;
        proxy_cache off;
        add_header X-Accel-Buffering no always;
    }
    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}


upstream backend {
     server backend-ap-internal.ilivedata.com:443;
 }
```
### tx-ap

```
upstream backend {
    server 127.0.0.1:8091 weight=10 max_fails=1 fail_timeout=5s;
}


server {
    listen 443 ssl http2;
    listen 80;
    server_name backend-ap-internal.ilivedata.com;
    client_max_body_size 1G;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 180s;
        proxy_send_timeout 60s;
        proxy_read_timeout 120s;
        proxy_pass      http://backend;
    }

    location /video/dubbing/generateAiRewriteStream {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 180s;
        proxy_send_timeout 60s;
        proxy_read_timeout 120s;
        proxy_pass      http://backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        chunked_transfer_encoding on;
        gzip off;
        proxy_cache off;
    }
}

server {
    listen 443 ssl http2;
    listen 80;
    server_name backend.ilivedata.com;
    client_max_body_size 1G;

    location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-Device-ID,X-App-Source,X-App-Version,X-App-Package';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-App-Source,X-App-Version,X-App-Package';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 180s;
        proxy_send_timeout 60s;
        proxy_read_timeout 120s;
        proxy_pass      http://backend;
    }
    location /video/dubbing/generateAiRewriteStream {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-Device-ID,X-App-Source,X-App-Version,X-App-Package';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'X-Channel,X-Visitor-Id,X-User-Id,Authorization,DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,X-App-Source,X-App-Version,X-App-Package';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 180s;
        proxy_send_timeout 60s;
        proxy_read_timeout 120s;
        proxy_pass      http://backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        chunked_transfer_encoding on;
        gzip off;
        proxy_cache off;
    }
}
```

## 效果验证
![](/blogs/resolve-cros/59feeaf16999d525.png)

## 总结
  在这个案例中，学习到了Host字段的详细作用，Nginx 在代理转发时，默认传递客户端原始的 Host 请求头。目标 Nginx 接收到请求后，根据 Host 头的值来匹配 server_name，从而决定由哪个 server 块处理。