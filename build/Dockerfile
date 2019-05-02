FROM arm32v7/alpine
MAINTAINER Juan Pablo Martin <jpmartin@gicom.com.ar>

# Important!  Update this no-op ENV variable when this Dockerfile
# is updated with the current date. It will force refresh of all
# of the base images and things like `apt-get update` won't be using
# old cached versions when the Dockerfile is built.

ENV REFRESHED_AT=2019-04-29 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    HOME=/opt/app/ \
    # Set this so that CTRL+G works properly
    TERM=xterm \
    DEBIAN_FRONTEND=noninteractive \
    ERLANG_VERSION=21.3 \
    EMQ_VERSION=v2.2-rc.1

ENV BUILD_PACKAGES \
      locales \
      build-essential \
      autoconf \
      libncurses5-dev \
      libssl-dev \
      curl \
      observer \
      ca-certificates \
      procps \
      git \
      memprof

WORKDIR /tmp/erlang-build

# Install Erlang
RUN \
    # Create default user and home directory, set owner to default
    mkdir -p ${HOME} && \
    adduser -s /bin/sh -u 1001 -G root -h ${HOME} -S -D default && \
    chown -R 1001:0 ${HOME} && \
    # Add tagged repos as well as the edge repo so that we can selectively install edge packages
#    echo "@main http://dl-cdn.alpinelinux.org/alpine/v3.7/main" >> /etc/apk/repositories && \
#    echo "@community http://dl-cdn.alpinelinux.org/alpine/v3.7/community" >> /etc/apk/repositories && \
#    echo "@edge http://nl.alpinelinux.org/alpine/edge/main" >> /etc/apk/repositories && \
    # Upgrade Alpine and base packages
    apk --no-cache upgrade && \
    # Distillery requires bash
    apk add --no-cache --virtual memprof && \
    apk add --no-cache --virtual .erlang-build \
    # Install Erlang/OTP deps
      bash \
      ca-certificates \
      openssl-dev \
      pcre \
      ncurses-dev \
      unixodbc-dev \
      zlib-dev \
      dpkg-dev dpkg \
      git autoconf build-base perl-dev \
      bsd-compat-headers libc-dev  memprof \
      git autoconf make gcc curl ncurses-libs  && \
    # Shallow clone Erlang/OTP
    git clone -b OTP-$ERLANG_VERSION --single-branch --depth 1 https://github.com/erlang/otp.git .  && \
    # Erlang/OTP build env
    export ERL_TOP=/tmp/erlang-build && \
    export PATH=$ERL_TOP/bin:$PATH && \
    export CPPFlAGS="-D_BSD_SOURCE $CPPFLAGS" && \
    # Configure
    ./otp_build autoconf && \
    ./configure --prefix=/usr \
      --build="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
      --sysconfdir=/etc \
      --mandir=/usr/share/man \
      --infodir=/usr/share/info \
      --without-javac \
      --without-wx \
      --without-debugger \
      --without-observer \
      --without-jinterface \
      --without-cosEvent\
      --without-cosEventDomain \
      --without-cosFileTransfer \
      --without-cosNotification \
      --without-cosProperty \
      --without-cosTime \
      --without-cosTransactions \
      --without-et \
      --without-gs \
      --without-ic \
      --without-megaco \
      --without-orber \
      --without-percept \
      --without-typer \
      --enable-threads \
      --enable-shared-zlib \
      --enable-ssl=dynamic-ssl-lib \
      --enable-hipe && \
    # Build
    make -j4 && make install && \
    # Update ca certificates
    update-ca-certificates --fresh


# Intalling Erlang Environment
# Based on https://github.com/joakimk/rpi-erlang

#WORKDIR ${HOME}
WORKDIR /opt/emqttd
COPY ./start.sh /start.sh

# Now compile EMQ Project
RUN git clone https://github.com/emqtt/emq-relx.git /emqttd && \
    cd /emqttd && sed -i 's/{observer, load},/ /g' relx.config && make && \
    mkdir -p /opt && mv /emqttd/_rel/emqx /opt/emqttd && \
    cd / && rm -rf /emqttd && \
    mv /start.sh /opt/emqttd/start.sh && \
    chmod +x /opt/emqttd/start.sh && \
    ln -s /opt/emqttd/bin/* /usr/local/bin/ && \

    # Cleanup
    apk del --force .erlang-build && \
    cd $HOME && \
    rm -rf /tmp/erlang-build

# remove some unused build deps to reduce the package size
RUN apk add remove -qq --purge -y $BUILD_PACKAGES \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/local/lib/erlang


#WORKDIR /opt/emqttd

# start emqttd and initial environments
CMD ["bash", "/opt/emqttd/start.sh"]

VOLUME ["/opt/emqttd/log", "/opt/emqttd/data", "/opt/emqttd/lib", "/opt/emqttd/etc"]

# emqttd will occupy these port:
# - 1883 port for MQTT
# - 8883 port for MQTT(SSL)
# - 8083 for WebSocket/HTTP
# - 8084 for WSS/HTTPS
# - 18083 for dashboard
# - 4369 for port mapping
# - 6000-6999 for distributed node
EXPOSE 1883 8883 8083 8084 18083 4369 6000-6999