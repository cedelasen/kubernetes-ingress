ARG GOLANG_CONTAINER=golang:latest
ARG NGINX_VERSION=1.19.1

#VTS Module stuff
FROM nginx:${NGINX_VERSION} AS helper

ENV VTS_VERSION=0.1.18
ENV VTS_GITREPO=vozlt/nginx-module-vts

# Install necessary dependencies
RUN apt-get update && \
    apt-get install --no-install-recommends --no-install-suggests -y \
    wget \
    gcc \
    make \
    libpcre3 \
    libpcre3-dev \
    libssl-dev \
    zlib1g \ 
    zlib1g-dev

# Download sources
RUN wget "http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" -O nginx.tar.gz && \
    wget "https://github.com/${VTS_GITREPO}/archive/v${VTS_VERSION}.tar.gz" -O extra_module.tar.gz

RUN rm -rf /usr/src/nginx /usr/src/extra_module && mkdir -p /usr/src/nginx /usr/src/extra_module && \
    tar -zxC /usr/src/nginx -f nginx.tar.gz && \
    tar -xzC /usr/src/extra_module -f extra_module.tar.gz

WORKDIR /usr/src/nginx/nginx-${NGINX_VERSION}

# Reuse same cli arguments as the nginx:debian image used to build
RUN CONFARGS=$(nginx -V 2>&1 | sed -n -e 's/^.*arguments: //p') && \
    sh -c "./configure --with-compat $CONFARGS --add-dynamic-module=/usr/src/extra_module/*" && make modules


FROM nginx:${NGINX_VERSION} AS base

# forward nginx access and error logs to stdout and stderr of the ingress
# controller process
RUN ln -sf /proc/1/fd/1 /var/log/nginx/access.log \
	&& ln -sf /proc/1/fd/1 /var/log/nginx/stream-access.log \
	&& ln -sf /proc/1/fd/2 /var/log/nginx/error.log

RUN mkdir -p /var/lib/nginx \
	&& mkdir -p /etc/nginx/secrets \
	&& mkdir -p /etc/nginx/stream-conf.d \
	&& apt-get update \
	&& apt-get install -y libcap2-bin \
	&& setcap 'cap_net_bind_service=+ep' /usr/sbin/nginx \
	&& setcap 'cap_net_bind_service=+ep' /usr/sbin/nginx-debug \
	&& chown -R nginx:0 /etc/nginx \
	&& chown -R nginx:0 /var/cache/nginx \
	&& chown -R nginx:0 /var/lib/nginx \
	&& apt-get remove --purge -y libcap2-bin \
	&& rm /etc/nginx/conf.d/* \
	&& rm -rf /var/lib/apt/lists/*

COPY internal/configs/version1/nginx.ingress.tmpl \
	internal/configs/version1/nginx.tmpl \
	internal/configs/version2/nginx.virtualserver.tmpl \
	internal/configs/version2/nginx.transportserver.tmpl /

# Uncomment the line below if you would like to add the default.pem to the image
# and use it as a certificate and key for the default server
# ADD default.pem /etc/nginx/secrets/default

USER nginx

ENTRYPOINT ["/nginx-ingress"]


FROM base AS local
COPY nginx-ingress /


FROM $GOLANG_CONTAINER AS builder
ARG VERSION
ARG GIT_COMMIT
WORKDIR /go/src/github.com/nginxinc/kubernetes-ingress/nginx-ingress/cmd/nginx-ingress
COPY . /go/src/github.com/nginxinc/kubernetes-ingress/nginx-ingress/
RUN CGO_ENABLED=0 GOFLAGS='-mod=vendor' \
	go build -installsuffix cgo -ldflags "-w -X main.version=${VERSION} -X main.gitCommit=${GIT_COMMIT}" -o /nginx-ingress

# Production container starts here
FROM base AS container
COPY --from=helper /usr/src/nginx/nginx-${NGINX_VERSION}/objs/*_module.so /etc/nginx/modules/
COPY --from=builder /nginx-ingress /
