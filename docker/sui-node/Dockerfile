ARG PROFILE=release
ARG BUILD_DATE
ARG GIT_REVISION
ARG RUST_VERSION=1.76.0

FROM scratch AS base

FROM docker.io/stagex/rust:${RUST_VERSION} AS rust
FROM base AS fetch

COPY --from=docker.io/stagex/busybox . /
COPY --from=docker.io/stagex/musl . /
COPY --from=rust . /

COPY --from=docker.io/stagex/gcc . /
COPY --from=docker.io/stagex/llvm . /
COPY --from=docker.io/stagex/libunwind . /
COPY --from=docker.io/stagex/openssl . /
COPY --from=docker.io/stagex/zlib . /

# NOTE: Necessary for `cargo fetch`, but CA trust is not relied upon
COPY --from=docker.io/stagex/ca-certificates . /

# HACK: gcc puts things in /usr/lib64 
COPY --from=docker.io/stagex/gcc /usr/lib64/* /usr/lib/

COPY . /sui

WORKDIR sui

RUN cargo fetch

FROM fetch AS build

# Rust build deps

COPY --from=docker.io/stagex/binutils . /
COPY --from=docker.io/stagex/gcc . /
COPY --from=docker.io/stagex/llvm . /
COPY --from=docker.io/stagex/make . /
COPY --from=docker.io/stagex/musl . /

# Sui build deps

COPY --from=docker.io/stagex/clang . /
COPY --from=docker.io/stagex/linux-headers . /

ARG PROFILE
ARG GIT_REVISION
ENV RUST_BACKTRACE=1
ENV RUSTFLAGS="-C target-feature=-crt-static -C codegen-units=1 -C passes=sancov-module \
-C llvm-args=-sanitizer-coverage-level=3 \
-C llvm-args=-sanitizer-coverage-trace-pc-guard \
-C link-args=-Wl,--build-id -Ccodegen-units=1 \
-L/usr/lib/libvoidstar.so -lvoidstar"
ENV GIT_REVISION=${GIT_REVISION}
ENV PROFILE=${PROFILE}

# Build step with Antithesis instrumentation
COPY /ant/libvoidstar.so /usr/lib/libvoidstar.so

RUN --network=none LD_LIBRARY_PATH=/usr/lib/libvoidstar.so cargo build --frozen --profile ${PROFILE} --bin sui-node

# RUN --network=none cargo build --frozen --profile ${PROFILE} --bin sui-node

FROM scratch AS install

COPY --from=stagex/busybox . /

COPY --from=docker.io/stagex/busybox . /rootfs
COPY --from=docker.io/stagex/libunwind . /rootfs
COPY --from=docker.io/stagex/gcc . /rootfs
COPY --from=docker.io/stagex/musl . /rootfs

# HACK: In the current release of stagex, gcc puts things in /usr/lib64,
# but we expect them in /usr/lib
COPY --from=docker.io/stagex/gcc /usr/lib64/* /rootfs/usr/lib/

# support current + legacy paths
RUN mkdir -p /rootfs/opt/sui/bin
RUN mkdir -p /rootfs/usr/local/bin
COPY --from=build sui/target/release/sui-node /rootfs/opt/sui/bin/sui-node



RUN --network=none find /rootfs -exec touch -hcd "@0" "{}" +

FROM scratch AS package

ARG PROFILE
ARG GIT_REVISION

LABEL build-date=${BUILD_DATE}
LABEL git-revision=${GIT_REVISION}

COPY --from=install /rootfs /

RUN ln -s /opt/sui/bin/sui-node /usr/local/bin/sui-node

