# Build stage for Syscoin Core
FROM alpine as syscoin-core

RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add autoconf
RUN apk --no-cache add automake
RUN apk --no-cache add boost-dev
RUN apk --no-cache add build-base
RUN apk --no-cache add chrpath
RUN apk --no-cache add file
RUN apk --no-cache add gnupg
RUN apk --no-cache add libevent-dev
RUN apk --no-cache add curl-dev
RUN apk --no-cache add gnutls-dev
RUN apk --no-cache add gmp-dev
RUN apk --no-cache add libtool
RUN apk --no-cache add linux-headers
RUN apk --no-cache add zeromq-dev
RUN apk --no-cache add cmake
RUN apk --no-cache add gcc
RUN apk --no-cache add g++
RUN apk --no-cache add libc-dev
RUN apk --no-cache add make
RUN apk --no-cache add git

RUN set -ex \
  && for key in \
    90C8019E36C2E964 \
  ; do \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
    gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" || \
    gpg --batch --keyserver keyserver.pgp.com --recv-keys "$key" || \
    gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" || \
    gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key" ; \
  done

ENV SYSCOIN_VERSION=4.3.99
ENV SYSCOIN_PREFIX=/opt/syscoin-${SYSCOIN_VERSION}

RUN wget https://github.com/syscoin/syscoin/archive/refs/heads/dev-4.x-nevm.tar.gz
RUN tar -xzf *.tar.gz

WORKDIR /syscoin-dev-4.x-nevm

RUN sed -i '/AC_PREREQ/a\AR_FLAGS=cr' src/univalue/configure.ac
RUN sed -i '/AX_PROG_CC_FOR_BUILD/a\AR_FLAGS=cr' src/secp256k1/configure.ac
RUN sed -i s:sys/fcntl.h:fcntl.h: src/compat.h
RUN ./autogen.sh
run rm /usr/lib/libsodium.so
RUN ./configure --prefix=${SYSCOIN_PREFIX} \
    --mandir=/usr/share/man \
    --disable-tests \
    --disable-bench \
    --disable-ccache \
    --disable-wallet \
    --with-gui=no \
    --with-utils \
    --with-libs \
    --with-daemon
RUN make
RUN make install
RUN strip ${SYSCOIN_PREFIX}/bin/syscoin-cli
RUN strip ${SYSCOIN_PREFIX}/bin/syscoin-tx
RUN strip ${SYSCOIN_PREFIX}/bin/syscoind
RUN strip ${SYSCOIN_PREFIX}/lib/libsyscoinconsensus.a
RUN strip ${SYSCOIN_PREFIX}/lib/libsyscoinconsensus.so.0.0.0

# Build stage for compiled artifacts
FROM alpine

LABEL maintainer.0="Jagdeep Sidhu (@realSidhuJag)"

RUN adduser -S syscoin
RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add \
  boost-filesystem \
  boost-system \
  boost-thread \
  libevent \
  libzmq \
  su-exec \
  libcurl

ENV SYSCOIN_DATA=/home/syscoin/.syscoin
ENV SYSCOIN_VERSION=4.3.99
ENV SYSCOIN_PREFIX=/opt/syscoin-${SYSCOIN_VERSION}
ENV PATH=${SYSCOIN_PREFIX}/bin:$PATH

COPY --from=syscoin-core /opt /opt
COPY docker-entrypoint.sh /entrypoint.sh

VOLUME ["/home/syscoin/.syscoin"]

EXPOSE 8369 8370 18369 18370 18443 18444 38332 38333

ENTRYPOINT ["/entrypoint.sh"]

RUN syscoind -version | grep "Syscoin Core version v${SYSCOIN_VERSION}"

CMD ["syscoind"]
