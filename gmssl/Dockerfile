FROM --platform=$BUILDPLATFORM debian:bookworm-slim AS builder
ARG TARGETARCH

ENV NGINX_VERSION 1.30.6

WORKDIR /data/builder/debuild

RUN set -eux; \
    build_arch="$(dpkg --print-architecture)"; \
    cross_compiler=''; \
    nginx_cc=cc; \
    if [ "$build_arch" != "$TARGETARCH" ]; then \
        dpkg --add-architecture "$TARGETARCH"; \
        case "$TARGETARCH" in \
            amd64) cross_compiler=gcc-x86-64-linux-gnu; nginx_cc=x86_64-linux-gnu-gcc ;; \
            arm64) cross_compiler=gcc-aarch64-linux-gnu; nginx_cc=aarch64-linux-gnu-gcc ;; \
            *) echo "Unsupported target architecture: $TARGETARCH" >&2; exit 1 ;; \
        esac; \
    fi; \
    apt-get update; \
    apt-get install --no-install-recommends --no-install-suggests -y \
        build-essential ca-certificates wget \
        "libpcre2-dev:$TARGETARCH" "zlib1g-dev:$TARGETARCH" $cross_compiler; \
    ln -s "/usr/bin/$nginx_cc" /usr/local/bin/nginx-cc; \
    rm -rf /var/lib/apt/lists/*

ARG SSL_NAMR=gmssl
ARG NGINX_CONFIGURE_ARGS="--prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module"

COPY gmssl/gmssl /usr/local/gmssl
RUN set -ex \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && addgroup --system --gid 101 nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false --uid 101 nginx \
    && wget https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
    && tar xf nginx-${NGINX_VERSION}.tar.gz \
    && sed -i 's@$OPENSSL/.openssl/@$OPENSSL/@g' nginx-${NGINX_VERSION}/auto/lib/openssl/conf \
    && sed -i 's/uname -m/nginx-cc -dumpmachine | cut -d- -f1/' nginx-${NGINX_VERSION}/configure \
    && cd nginx-${NGINX_VERSION} \
    && ./configure ${NGINX_CONFIGURE_ARGS} --with-cc=nginx-cc --with-openssl=/usr/local/${SSL_NAMR} --with-cc-opt=-I/usr/local/${SSL_NAMR}/include --with-ld-opt=-lm \
    && make -j$(nproc) \
    && make install \
    && mkdir -p /usr/share/nginx /var/cache/nginx \
    && mv /etc/nginx/html /usr/share/nginx \
    && rm -f /etc/nginx/nginx.conf /etc/nginx/*.default \
    && echo "no" | dpkg-reconfigure dash \
    && sed -i "s@# export @export @g" ~/.bashrc \
    && sed -i "s@# alias @alias @g" ~/.bashrc \
    && rm -rf /var/lib/apt/lists/*

COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d /etc/nginx/conf.d

FROM debian:bookworm-slim
ARG TARGETARCH

COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /etc/nginx /etc/nginx
COPY --from=builder /usr/share/nginx /usr/share/nginx

RUN set -ex \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && addgroup --system --gid 101 nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false --uid 101 nginx \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y curl ca-certificates libpcre2-8-0 zlib1g \
    && mkdir -p /var/cache/nginx /var/log/nginx \
    && nginx -t \
    && echo "no" | dpkg-reconfigure dash \
    && sed -i "s@# export @export @g" ~/.bashrc \
    && sed -i "s@# alias @alias @g" ~/.bashrc \
    && apt-get purge --auto-remove -y \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 80

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
