# 构建说明

四个镜像默认使用 Nginx **1.30.6**（稳定版）和 Debian Trixie，保留各自的国密 SSL 库。
GmSSL、GDCA 和 WoTrus 构建阶段使用本机架构，跨架构构建时安装目标架构的交叉编译器；配置检测和运行检查仍需 QEMU/Rosetta 支持。
铜锁 8.4.0 镜像暂不支持 `ssl_certificate_compression`。
Nginx 源码通过 HTTPS 下载，构建时会执行 `nginx -t` 检查默认配置和运行依赖。

在仓库根目录构建，例如：

```sh
docker build --platform linux/amd64 -f tongsuo/Dockerfile -t tongsuo_nginx:1.30.6 .
docker build --platform linux/amd64 -f gmssl/Dockerfile -t gmssl_nginx:1.30.6 .
docker build --platform linux/amd64 -f trustauth/Dockerfile -t trustauth_nginx:1.30.6 .
docker build --platform linux/amd64 -f wotrus/Dockerfile -t wotrus_nginx:1.30.6 .
```

仓库自带的开源 GmSSL 二进制库和 GDCA 库仅支持 Linux/amd64；WoTrus 支持 amd64、arm64，铜锁从源码构建。
可通过 `--build-arg NGINX_VERSION=版本号` 指定版本，手动构建工作流会将输入版本传给镜像构建。

# Tongsuo

- [铜锁 / Tongsuo](https://www.tongsuo.net/)

铜锁开源社区目前不直接提供有偿的商业服务, 可以通过 [天威诚信](https://www.itrus.com.cn/)、问天量子、[阿里云](https://www.aliyun.com/)、BTQ、蚂蚁集团 等生态合作方采购数字证书

运行容器
```sh
docker run --name nginx -d --restart=always \
  -p 80:80 -p 443:443 \
  -v /opt/sslkey:/etc/nginx/sslkey \
  -v /opt/default.conf:/etc/nginx/conf.d/default.conf \
  wojiushixiaobai/tongsuo_nginx:latest
```

# WoTrus 证书使用说明

- [沃通 SSL 证书](https://www.wosign.com/Docdownload/)

获取 ssl 证书后解压到服务器
```sh
ll /opt/sslkey
```
```sh
总用量 12
-rw-r--r--. 1 root root 3048 12月  2 19:07 test.domain.localhost_sm2_encrypt_bundle.crt
-rw-r--r--. 1 root root  227 12月  2 19:07 test.domain.localhost_SM2.key
-rw-r--r--. 1 root root 3048 12月  2 19:07 test.domain.localhost_sm2_sign_bundle.crt
```

生成 nginx 配置文件
```sh
vi /opt/default.conf
```

```nginx
server {
    listen 80;
    server_name test.domain.localhost;  # 自行修改成你的域名
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name test.domain.localhost;  # 自行修改成你的域名

    ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers ECC-SM4-SM3:ECDH:AESGCM:HIGH:MEDIUM:!RC4:!DH:!MD5:!aNULL:!eNULL;
    ssl_verify_client off;
    ssl_session_timeout 5m;
    ssl_prefer_server_ciphers on;

    ssl_certificate sslkey/test.domain.localhost_sm2_sign_bundle.crt;      # 配置国密签名证书/私钥
    ssl_certificate_key sslkey/test.domain.localhost_SM2.key;

    ssl_certificate sslkey/test.domain.localhost_sm2_encrypt_bundle.crt;   # 配置国密加密证书/私钥
    ssl_certificate_key sslkey/test.domain.localhost_SM2.key;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

运行容器
```sh
docker run --name nginx -d --restart=always \
  -p 80:80 -p 443:443 \
  -v /opt/sslkey:/etc/nginx/sslkey \
  -v /opt/default.conf:/etc/nginx/conf.d/default.conf \
  wojiushixiaobai/wotrus_nginx:latest
```

# GDCA 证书使用说明

[GDCA 数安时代](https://www.trustauth.cn/)

```sh
ll /opt/sslkey
```
```sh
总用量 12
-rw-r--r--. 1 root root 3048 12月  2 19:07 test.domain.localhost_encrypt_chain.crt
-rw-r--r--. 1 root root  227 12月  2 19:07 test.domain.localhost.key
-rw-r--r--. 1 root root 3048 12月  2 19:07 test.domain.localhost_sign_chain.crt
```

```sh
vi /opt/default.conf
```

```nginx
server {
    listen 80;
    server_name test.domain.localhost;  # 自行修改成你的域名
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name test.domain.localhost;  # 自行修改成你的域名

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-SM2-SM4-SM3:ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!3DES:!aNULL:!MD5:!ADH:!RC4;
    ssl_verify_client off;
    ssl_session_timeout 5m;
    ssl_prefer_server_ciphers on;

    ssl_certificate sslkey/test.domain.localhost_sign_chain.crt;      # 配置国密签名证书/私钥
    ssl_certificate_key sslkey/test.domain.localhost.key;

    ssl_certificate sslkey/test.domain.localhost_encrypt_chain.crt;   # 配置国密加密证书/私钥
    ssl_certificate_key sslkey/test.domain.localhost.key;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

```sh
docker run --name nginx -d --restart=always \
  -p 80:80 -p 443:443 \
  -v /opt/sslkey:/etc/nginx/sslkey \
  -v /opt/default.conf:/etc/nginx/conf.d/default.conf \
  wojiushixiaobai/trustauth_nginx:latest
```

# GMSSL 证书使用说明

[国密SSL guanzhi/GmSSL](https://github.com/guanzhi/GmSSL/tree/GmSSL-v2)
使用 v2 版本，只支持 TLS 1.2

```sh
ll /opt/sslkey
```

```sh
总用量 72
-rw-r--r--. 1 root root    8 12月  2 16:33 password.txt
-rw-r--r--. 1 root root 3161 12月  2 16:33 sm2.test.domain.localhost.both.pfx
-rw-r--r--. 1 root root  525 12月  2 16:33 sm2.test.domain.localhost.enc.cer
-rw-r--r--. 1 root root  765 12月  2 16:33 sm2.test.domain.localhost.enc.crt.pem
-rw-r--r--. 1 root root  150 12月  2 16:33 sm2.test.domain.localhost.enc.key
-rw-r--r--. 1 root root  520 12月  2 16:33 sm2.test.domain.localhost.enc.key.crypted.b64
-rw-r--r--. 1 root root  388 12月  2 16:33 sm2.test.domain.localhost.enc.key.crypted.bin
-rw-r--r--. 1 root root  227 12月  2 16:33 sm2.test.domain.localhost.enc.key.p8
-rw-r--r--. 1 root root  258 12月  2 16:33 sm2.test.domain.localhost.enc.key.pem
-rw-r--r--. 1 root root  962 12月  2 16:33 sm2.test.domain.localhost.enc.pfx
-rw-r--r--. 1 root root  524 12月  2 16:33 sm2.test.domain.localhost.sig.cer
-rw-r--r--. 1 root root  765 12月  2 16:33 sm2.test.domain.localhost.sig.crt.pem
-rw-r--r--. 1 root root  150 12月  2 16:33 sm2.test.domain.localhost.sig.key
-rw-r--r--. 1 root root  227 12月  2 16:33 sm2.test.domain.localhost.sig.key.p8
-rw-r--r--. 1 root root  258 12月  2 16:33 sm2.test.domain.localhost.sig.key.pem
-rw-r--r--. 1 root root  961 12月  2 16:33 sm2.test.domain.localhost.sig.pfx
-rw-r--r--. 1 root root  725 12月  2 16:33 sm2.oca.pem
-rw-r--r--. 1 root root  684 12月  2 16:33 sm2.rca.pem
```

```sh
vi /opt/default.conf
```

```nginx
server {
    listen 80;
    server_name test.domain.localhost;  # 自行修改成你的域名
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name test.domain.localhost;  # 自行修改成你的域名

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:AES128-SHA:DES-CBC3-SHA:ECC-SM4-SM3:ECDHE-SM4-SM3:SM2-WITH-SMS4-SM3:ECDHE-SM2-WITH-SMS4-GCM-SM3:ECDH:AESGCM:HIGH:MEDIUM:!RC4:!DH:!MD5:!aNULL:!eNULL;
    ssl_verify_client off;

    ssl_certificate sslkey/sm2.test.domain.localhost.sig.crt.pem;      # 配置国密签名证书/私钥
    ssl_certificate_key sslkey/sm2.test.domain.localhost.sig.key.pem;

    ssl_certificate_key sslkey/sm2.test.domain.localhost.enc.key.pem;  # 配置国密加密证书/私钥
    ssl_certificate sslkey/sm2.test.domain.localhost.enc.crt.pem;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

```sh
docker run --name nginx -d --restart=always \
  -p 80:80 -p 443:443 \
  -v /opt/sslkey:/etc/nginx/sslkey \
  -v /opt/default.conf:/etc/nginx/conf.d/default.conf \
  wojiushixiaobai/gmssl_nginx:latest
```
