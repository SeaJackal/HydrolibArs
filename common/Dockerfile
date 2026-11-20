ARG DEBIAN_VERSION=trixie-slim
ARG LLVM_VERSION=23.1.5
ARG CMAKE_VERSION=4.4.5

FROM --platform=$BUILDPLATFORM debian:${DEBIAN_VERSION} AS fetch

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        aria2 \
        ca-certificates \
        zstd \
    && rm -rf /var/lib/apt/lists/*

FROM fetch AS llvm

ARG LLVM_VERSION

RUN --mount=type=cache,target=/var/cache/downloads,sharing=locked \
    set -eu; \
    archive="LLVM-${LLVM_VERSION}-Linux-X64.tar.zst"; \
    cd /var/cache/downloads; \
    if ! test -f "$archive" || test -f "$archive.aria2"; then \
        aria2c --allow-overwrite=true --auto-file-renaming=false \
            --max-connection-per-server=8 --split=8 \
            --file-allocation=none --retry-wait=3 \
            --show-console-readout=false \
            --out="$archive" \
            "https://github.com/llvm/llvm-project/releases/download/llvmorg-${LLVM_VERSION}/${archive}"; \
    fi; \
    mkdir /tmp/llvm; \
    tar --use-compress-program='zstd --long=31' -xf "$archive" -C /tmp/llvm --strip-components=1; \
    mkdir -p /opt/llvm/bin "/opt/llvm/lib/clang/${LLVM_VERSION%%.*}"; \
    cp -a /tmp/llvm/bin/clang-format /tmp/llvm/bin/clang-tidy /tmp/llvm/bin/clangd /opt/llvm/bin/; \
    cp -a "/tmp/llvm/lib/clang/${LLVM_VERSION%%.*}/include" "/opt/llvm/lib/clang/${LLVM_VERSION%%.*}/"; \
    for license in /tmp/llvm/LICENSE*; do \
        if [ -f "$license" ]; then cp -a "$license" /opt/llvm/; fi; \
    done; \
    rm -rf /tmp/llvm

FROM fetch AS cmake

ARG CMAKE_VERSION

RUN --mount=type=cache,target=/var/cache/cmake,sharing=locked \
    set -eu; \
    archive="cmake-${CMAKE_VERSION}-linux-x86_64.tar.gz"; \
    cd /var/cache/cmake; \
    if ! test -f "$archive" || test -f "$archive.aria2"; then \
        aria2c --allow-overwrite=true --auto-file-renaming=false \
            --max-connection-per-server=8 --split=8 \
            --file-allocation=none --retry-wait=3 \
            --show-console-readout=false \
            --out="$archive" \
            "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/${archive}"; \
    fi; \
    mkdir /opt/cmake; \
    tar -xzf "$archive" -C /opt/cmake --strip-components=1

FROM --platform=linux/amd64 debian:${DEBIAN_VERSION} AS common

ARG USERNAME=hydrolib
ARG UID=1000
ARG GID=$UID

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        bash \
        bash-completion \
        ca-certificates \
        git \
        libgcc-s1 \
        libstdc++6 \
        make \
        ninja-build \
        openssh-client \
        sudo \
        zlib1g \
    && rm -rf /var/lib/apt/lists/*

COPY --from=llvm /opt/llvm/ /opt/llvm/
COPY --from=cmake /opt/cmake/ /opt/cmake/

ENV PATH="/opt/cmake/bin:/opt/llvm/bin:${PATH}" \
    LANG=C.UTF-8

RUN for tool in cmake ctest cpack; do \
        ln -sf "/opt/cmake/share/bash-completion/completions/$tool" "/usr/share/bash-completion/completions/$tool"; \
    done \
    && clang-format --version \
    && clang-tidy --version \
    && clangd --version \
    && cmake --version \
    && ctest --version \
    && cpack --version

RUN groupadd --gid "$GID" "$USERNAME" \
    && useradd --uid "$UID" --gid "$GID" --create-home --shell /bin/bash "$USERNAME" \
    && printf '%s ALL=(ALL) NOPASSWD:ALL\n' "$USERNAME" > "/etc/sudoers.d/$USERNAME" \
    && chmod 0440 "/etc/sudoers.d/$USERNAME" \
    && visudo --check \
    && mkdir /workspaces \
    && chown "$UID:$GID" /workspaces

USER $USERNAME
WORKDIR /workspaces
CMD ["sleep", "infinity"]
