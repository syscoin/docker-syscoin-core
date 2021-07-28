# Build Geth in a stock Go builder container
FROM golang:1.16-alpine as go-ethereum

ARG GETH_REPO=https://github.com/sidhujag/go-ethereum.git
ARG GETH_BRANCH=nevm

RUN set -xe; \
  apk add --no-cache --virtual .build-deps \
  make gcc musl-dev linux-headers git; \
  mkdir -p go-ethereum; \
  git clone ${GETH_REPO} --branch ${GETH_BRANCH} --depth 1 --single-branch go-ethereum; \
  cd go-ethereum; \
  make geth -j$(nproc); \
  apk del .build-deps;

FROM alpine:3.14 as syscoin-build

ARG SYSCOIN_VERSION=4.3.99
ARG SYSCOIN_PREFIX=/opt/syscoin-${SYSCOIN_VERSION}
ARG SYSCOIN_BRANCH=dev-4.x-nevm
ARG GPG_KEY="90C8019E36C2E964"

# Build Syscoin Core
RUN set set -xe; \
  # use ssl repos
  sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories; \
  # build dependencies
  apk add --no-cache --virtual .build-deps \
  autoconf autoconf automake build-base chrpath file gnupg libtool linux-headers \
  cmake libc-dev make git python3 xz bison curl pkgconf bash clang; \
  # validate key
  for key in ${GPG_KEY}; do \
  gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
  gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" || \
  gpg --batch --keyserver keyserver.pgp.com --recv-keys "$key" || \
  gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" || \
  gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key" ; \
  done; \
  # fetch nevm
  git clone https://github.com/syscoin/syscoin.git --branch ${SYSCOIN_BRANCH} --depth 1 --single-branch syscoin; cd syscoin; \
  # setup
  sed -i '/AC_PREREQ/a\AR_FLAGS=cr' src/univalue/configure.ac; \
  sed -i '/AX_PROG_CC_FOR_BUILD/a\AR_FLAGS=cr' src/secp256k1/configure.ac; \
  sed -i s:sys/fcntl.h:fcntl.h: src/compat.h; \
  # compile dependencies
  make -C depends -j $(nproc);\
  # configure
  ./autogen.sh; \
  CONFIG_SITE=$PWD/depends/x86_64-pc-linux-musl/share/config.site ./configure \
  --prefix=${SYSCOIN_PREFIX} \
  --mandir=/usr/share/man \
  --disable-tests \
  --disable-bench \
  --disable-ccache \
  --disable-maintainer-mode \
  --disable-dependency-tracking \
  --enable-reduce-exports \
  --with-gui=no \
  --with-utils \
  --with-libs \
  --with-daemon \
  CC=clang CXX=clang++ \
  CFLAGS="-O2 -g0 --static -static -fPIC" \
  CXXFLAGS="-O2 -g0 --static -static -fPIC" \
  LDFLAGS="-s -static-libgcc -static-libstdc++ -Wl,-O2"; \
  # compile
  make install -j $(nproc); \
  strip ${SYSCOIN_PREFIX}/bin/syscoin-cli; \
  strip ${SYSCOIN_PREFIX}/bin/syscoin-tx; \
  strip ${SYSCOIN_PREFIX}/bin/syscoind; \
  # remove build-deps
  apk del .build-deps; \
  # add run user
  adduser -S syscoin; \
  apk --no-cache add su-exec ca-certificates; \
  # validate install
  ${SYSCOIN_PREFIX}/bin/syscoind -version | grep "Syscoin Core version v${SYSCOIN_VERSION}"

FROM alpine:3.14 as syscoin-core

ARG SYSCOIN_VERSION=4.3.99
ARG SYSCOIN_PREFIX=/opt/syscoin-${SYSCOIN_VERSION}

LABEL org.label-schema.schema-version = "1.0"
LABEL org.label-schema.vendor = "Syscoin Core"
LABEL org.label-schema.version = ${SYSCOIN_VERSION}
LABEL org.label-schema.license = "MIT"
LABEL maintainer="Jagdeep Sidhu (@realSidhuJag), Justin Silver (@justinmsilver)"

ENV SYSCOIN_DATA=/home/syscoin/.syscoin
ENV SYSCOIN_VERSION=${SYSCOIN_VERSION}
ENV SYSCOIN_PREFIX=${SYSCOIN_PREFIX}
ENV LC_ALL C

RUN set set -xe; \
    mkdir -p ${SYSCOIN_DATA}; \
    apk add --no-cache wget; \
    wget https://raw.githubusercontent.com/syscoin/descriptors/master/gethdescriptor.json -O ${SYSCOIN_DATA}/gethdescriptor.json; \
    chmod +x ${SYSCOIN_DATA}/gethdescriptor.json

# Copy NEVM binary to the container
COPY --from=go-ethereum /go/go-ethereum/build/bin/geth ${SYSCOIN_DATA}/sysgeth
# Copy NEVM binary to the container
COPY --from=syscoin-build ${SYSCOIN_PREFIX}/bin/* /usr/local/bin/

COPY docker-entrypoint.sh /docker-entrypoint.sh

VOLUME ["/home/syscoin/.syscoin"]

EXPOSE 8369 8370 18369 18370 18443 18444 38332 38333

ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["syscoind"]