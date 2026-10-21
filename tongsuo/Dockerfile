FROM debian:trixie-slim AS builder
ARG TARGETARCH

ARG NGINX_VERSION=1.30.6
ENV NGINX_VERSION=${NGINX_VERSION}

WORKDIR /data/builder/debuild

ARG SSL_NAMR=tongsuossl
ARG NGINX_CONFIGURE_ARGS="--prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module"

RUN set -ex \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && groupadd --system --gid 101 nginx \
    && useradd --system --gid nginx --no-create-home --home /nonexistent --comment "nginx user" --shell /bin/false --uid 101 nginx \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y build-essential ca-certificates libpcre2-dev wget zlib1g-dev \
    && SSL_URL=https://codeload.github.com/Tongsuo-Project/Tongsuo/tar.gz/refs/tags/8.4.0 \
    && mkdir -p ${SSL_NAMR} \
    && wget ${SSL_URL} -O ${SSL_NAMR}.tar.gz \
    && tar xf ${SSL_NAMR}.tar.gz --strip-components=1 -C ${SSL_NAMR} \
    && cd ${SSL_NAMR} \
    && ./config no-shared no-tests --prefix=/usr/local/${SSL_NAMR} --libdir=lib \
    && make -j$(nproc) \
    && make install

# Tongsuo 8.4 exposes compression IDs without the BoringSSL CBB API.
RUN set -ex \
    && wget https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
    && tar xf nginx-${NGINX_VERSION}.tar.gz \
    && sed -i 's@$OPENSSL/.openssl/@$OPENSSL/@g' nginx-${NGINX_VERSION}/auto/lib/openssl/conf \
    && sed -i 's/defined TLSEXT_cert_compression_zlib/defined OPENSSL_IS_BORINGSSL \&\& defined TLSEXT_cert_compression_zlib/g' nginx-${NGINX_VERSION}/src/event/ngx_event_openssl.c \
    && cd nginx-${NGINX_VERSION} \
    && ./configure ${NGINX_CONFIGURE_ARGS} --with-openssl="/usr/local/${SSL_NAMR}" --with-cc-opt="-I/usr/local/${SSL_NAMR}/include" --with-ld-opt='-lm' \
    && make -j$(nproc) \
    && make install \
    && mkdir -p /usr/share/nginx /var/cache/nginx \
    && mv /etc/nginx/html /usr/share/nginx \
    && rm -f /etc/nginx/nginx.conf /etc/nginx/*.default \
    && rm -rf /var/lib/apt/lists/*

COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d /etc/nginx/conf.d

FROM debian:trixie-slim
ARG TARGETARCH

COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /etc/nginx /etc/nginx
COPY --from=builder /usr/share/nginx /usr/share/nginx

RUN set -ex \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && groupadd --system --gid 101 nginx \
    && useradd --system --gid nginx --no-create-home --home /nonexistent --comment "nginx user" --shell /bin/false --uid 101 nginx \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y curl ca-certificates libpcre2-8-0 zlib1g \
    && mkdir -p /var/cache/nginx /var/log/nginx \
    && nginx -t \
    && apt-get purge --auto-remove -y \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 80

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
