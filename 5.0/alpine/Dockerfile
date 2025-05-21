# Build Geth (Syscoin NEVM)
FROM golang:1.24-alpine AS go-ethereum
ARG GETH_REPO=https://github.com/syscoin/go-ethereum.git
ARG TAG=v5.0.3
RUN apk add --no-cache gcc musl-dev linux-headers git

RUN git clone --depth 1 --branch ${TAG} ${GETH_REPO} /go-ethereum
WORKDIR /go-ethereum
RUN go mod download
RUN go run build/ci.go install -static ./cmd/geth

# Build Syscoin Core
FROM alpine as syscoin-core
RUN sed -i 's/http:\/\/dl-cdn.alpinelinux.org/https:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add \
    autoconf \
    automake \
    boost-dev \
    build-base \
    chrpath \
    file \
    gnupg \
    libevent-dev \
    libtool \
    linux-headers \
    zeromq-dev \
    git \
    python3 \
    xz \
    bison \
    curl \
    pkgconf \
    bash \
    clang \
    util-macros \
    gmp-dev \
    sqlite-dev \
    cmake

ENV SYSCOIN_PREFIX=/opt/syscoin
ARG SYS_REPO=https://github.com/syscoin/syscoin.git
ARG TAG=v5.0.4
RUN git clone --depth 1 --branch ${TAG} ${SYS_REPO} /syscoin
#RUN git clone ${SYS_REPO} /syscoin
WORKDIR /syscoin
RUN ./autogen.sh && ./configure \
    --prefix=${SYSCOIN_PREFIX} \
    --disable-tests \
    --disable-bench \
    --disable-ccache \
    --disable-fuzz-binary \
    --without-utils \
    --with-gui=no \
    --enable-util-cli \
    --with-libs \
    --with-daemon
RUN make -j$(nproc) install
RUN strip ${SYSCOIN_PREFIX}/bin/syscoind ${SYSCOIN_PREFIX}/bin/syscoin-cli
RUN rm ${SYSCOIN_PREFIX}/bin/sysgeth
# Final runtime stage
FROM alpine:latest

LABEL maintainer.0="Jagdeep Sidhu (@realSidhuJag)"

RUN sed -i 's/http:\/\/dl-cdn.alpinelinux.org/https:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add \
  boost-filesystem \
  boost-system \
  boost-thread \
  libevent \
  libzmq \
  su-exec \
  ca-certificates \
  gmp \
  sqlite-dev \
  bash

RUN adduser -S syscoin

ENV SYSCOIN_PREFIX=/opt/syscoin
ENV PATH=${SYSCOIN_PREFIX}/bin:$PATH
ENV SYSCOIN_VERSION=5.0.4

# Copy binaries from build stages
COPY --from=syscoin-core /opt/syscoin /opt/syscoin
COPY --from=go-ethereum /go-ethereum/build/bin/geth ${SYSCOIN_PREFIX}/bin/sysgeth

COPY docker-entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER syscoin

VOLUME ["/home/syscoin/.syscoin"]

EXPOSE 8369 8370 18369 18370 18443 18444 38332 38333 8545 8546 30303 30303/udp

ENTRYPOINT ["/entrypoint.sh"]
RUN syscoind -version | grep "Syscoin Core version v${SYSCOIN_VERSION}"
CMD ["syscoind"]