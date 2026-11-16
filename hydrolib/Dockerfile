ARG DEBIAN_VERSION=trixie-slim
ARG GCC_VERSION=16.2.0
ARG LLVM_VERSION=23.1.5
ARG CMAKE_VERSION=4.4.5
ARG LCOV_VERSION=2.6
ARG COMPDB_VERSION=0.2.0

FROM --platform=linux/amd64 gcc:${GCC_VERSION}-trixie AS gcc-source

FROM --platform=$BUILDPLATFORM debian:${DEBIAN_VERSION} AS fetch

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        aria2 \
        ca-certificates \
        curl \
        make \
        zstd \
    && rm -rf /var/lib/apt/lists/*

FROM fetch AS gcc

COPY --from=gcc-source /usr/local/ /opt/gcc/

RUN find /opt/gcc -depth \
        \( -name '*gfortran*' -o -name '*gccgo*' -o -name 'go' \
        -o -name 'gofmt' -o -name 'go1' -o -name 'f951' \
        -o -name 'finclude' -o -name 'libgo.*' -o -name 'libgobegin.*' \
        -o -name 'libcaf_single*' \) \
        -exec rm -rf {} + \
    && rm -rf /opt/gcc/share/man /opt/gcc/share/info /opt/gcc/share/locale

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
        aria2c --continue=true --allow-overwrite=true --auto-file-renaming=false \
            --max-connection-per-server=8 --split=8 \
            --file-allocation=none --retry-wait=3 \
            --show-console-readout=false \
            --out="$archive" \
            "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/${archive}"; \
    fi; \
    mkdir /opt/cmake; \
    tar -xzf "$archive" -C /opt/cmake --strip-components=1

FROM fetch AS lcov

ARG LCOV_VERSION

RUN curl --fail --silent --show-error --location --retry 3 \
        "https://github.com/linux-test-project/lcov/releases/download/v${LCOV_VERSION}/lcov-${LCOV_VERSION}.tar.gz" \
        --output /tmp/lcov.tar.gz \
    && mkdir /tmp/lcov \
    && tar -xzf /tmp/lcov.tar.gz -C /tmp/lcov --strip-components=1 \
    && make -C /tmp/lcov install PREFIX=/opt/lcov LCOV_NO_DOC=1 LCOV_NO_XS=1 \
    && rm -rf /tmp/lcov /tmp/lcov.tar.gz /opt/lcov/share/lcov/tests

FROM --platform=linux/amd64 debian:${DEBIAN_VERSION} AS development

ARG COMPDB_VERSION
ARG USERNAME=hydrolib
ARG UID=1000
ARG GID=$UID

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        bash \
        bash-completion \
        binutils \
        ca-certificates \
        doxygen \
        gdb \
        git \
        libatomic1 \
        libc6-dev \
        libcapture-tiny-perl \
        libdatetime-perl \
        libdevel-stacktrace-perl \
        libgtest-dev \
        libjson-xs-perl \
        libtimedate-perl \
        libzstd1 \
        make \
        ninja-build \
        openssh-client \
        perl \
        python3 \
        python3-venv \
        sudo \
        zlib1g \
    && rm -rf /var/lib/apt/lists/*

COPY --from=gcc /opt/gcc/ /usr/local/
COPY --from=llvm /opt/llvm/ /opt/llvm/
COPY --from=cmake /opt/cmake/ /opt/cmake/
COPY --from=lcov /opt/lcov/ /opt/lcov/

ENV PATH="/opt/cmake/bin:/opt/llvm/bin:/opt/lcov/bin:/opt/compdb/bin:${PATH}" \
    CC=/usr/local/bin/gcc \
    CXX=/usr/local/bin/g++ \
    LANG=C.UTF-8

RUN printf '%s\n' /usr/local/lib64 /usr/local/lib > /etc/ld.so.conf.d/000-gcc.conf \
    && ldconfig \
    && mkdir -p /etc/gdb/gdbinit.d \
    && printf '%s\n' \
        'add-auto-load-safe-path /usr/local/lib' \
        'add-auto-load-safe-path /usr/local/lib64' \
        > /etc/gdb/gdbinit.d/gcc.gdb \
    && ln -sf /usr/local/bin/gcc /usr/local/bin/cc \
    && ln -sf /usr/local/bin/g++ /usr/local/bin/c++ \
    && for tool in cmake ctest cpack; do \
        ln -sf "/opt/cmake/share/bash-completion/completions/$tool" "/usr/share/bash-completion/completions/$tool"; \
    done \
    && python3 -m venv /opt/compdb \
    && /opt/compdb/bin/pip install --no-cache-dir --only-binary=:all: "compdb==$COMPDB_VERSION" \
    && gcc --version \
    && g++ --version \
    && gcov --version \
    && clang-format --version \
    && clang-tidy --version \
    && clangd --version \
    && cmake --version \
    && lcov --version \
    && compdb --help

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
