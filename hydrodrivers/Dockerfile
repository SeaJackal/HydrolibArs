ARG DEBIAN_VERSION=trixie-slim
ARG COMMON_IMAGE=seajackal/hydrolib-common:latest
ARG GCC_ARM_VERSION=15.3.rel2
ARG RENODE_VERSION=1.17.0
ARG COMPDB_VERSION=0.2.0
ARG PYSERIAL_VERSION=3.5
ARG PYRENODE_REVISION=27362def38ce409b95c972ce494cd8517dac17a4

FROM --platform=$BUILDPLATFORM debian:${DEBIAN_VERSION} AS fetch

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        aria2 \
        ca-certificates \
        xz-utils \
    && rm -rf /var/lib/apt/lists/*

FROM fetch AS arm-toolchain

ARG GCC_ARM_VERSION

RUN --mount=type=cache,target=/var/cache/arm-toolchain,sharing=locked \
    set -eu; \
    archive="arm-gnu-toolchain-${GCC_ARM_VERSION}-x86_64-arm-none-eabi.tar.xz"; \
    cd /var/cache/arm-toolchain; \
    if ! test -f "$archive" || test -f "$archive.aria2"; then \
        aria2c --allow-overwrite=true --auto-file-renaming=false \
            --max-connection-per-server=8 --split=8 \
            --file-allocation=none --retry-wait=3 \
            --show-console-readout=false \
            --out="$archive" \
            "https://gitlab.arm.com/api/v4/projects/tooling%2Fgnu-toolchains-for-arm/packages/generic/gnu-toolchain/${GCC_ARM_VERSION}/${archive}"; \
    fi; \
    mkdir /opt/arm-none-eabi; \
    tar -xJf "$archive" -C /opt/arm-none-eabi --strip-components=1

FROM fetch AS renode

ARG RENODE_VERSION

RUN --mount=type=cache,target=/var/cache/renode,sharing=locked \
    set -eu; \
    archive="renode-${RENODE_VERSION}.linux-portable.tar.gz"; \
    cd /var/cache/renode; \
    if ! test -f "$archive" || test -f "$archive.aria2"; then \
        aria2c --allow-overwrite=true --auto-file-renaming=false \
            --max-connection-per-server=8 --split=8 \
            --file-allocation=none --retry-wait=3 \
            --show-console-readout=false \
            --out="$archive" \
            "https://github.com/renode/renode/releases/download/v${RENODE_VERSION}/${archive}"; \
    fi; \
    mkdir /opt/renode; \
    tar -xzf "$archive" -C /opt/renode --strip-components=1

FROM --platform=linux/amd64 ${COMMON_IMAGE} AS development

ARG COMPDB_VERSION
ARG PYSERIAL_VERSION
ARG PYRENODE_REVISION
ARG USERNAME=hydrolib

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gdb-multiarch \
        libicu76 \
        libssl3t64 \
        openocd \
        python3 \
        python3-venv \
    && rm -rf /var/lib/apt/lists/*

COPY --from=arm-toolchain /opt/arm-none-eabi/ /opt/arm-none-eabi/
COPY --from=renode --chown=$USERNAME /opt/renode/ /opt/renode/

ENV PATH="/opt/arm-none-eabi/bin:/opt/renode:/opt/hydrodrivers-python/bin:${PATH}" \
    PYRENODE_PATH=/opt/renode

RUN python3 -m venv /opt/hydrodrivers-python \
    && /opt/hydrodrivers-python/bin/pip install --no-cache-dir \
        "compdb==$COMPDB_VERSION" \
        "pyserial==$PYSERIAL_VERSION" \
        "pyrenode3[all] @ git+https://github.com/antmicro/pyrenode3.git@$PYRENODE_REVISION" \
    && usermod --append --groups dialout,plugdev "$USERNAME" \
    && arm-none-eabi-gcc --version \
    && arm-none-eabi-g++ --version \
    && arm-none-eabi-size --version \
    && gdb-multiarch --version \
    && openocd --version \
    && cmake --version \
    && clang-format --version \
    && clang-tidy --version \
    && clangd --version \
    && compdb --help

USER $USERNAME

RUN renode --disable-gui --version \
    && python3 -c 'import serial; import pyrenode3'
