ARG DEBIAN_VERSION=trixie-slim
ARG COMMON_IMAGE=seajackal/hydrolib-common:latest
ARG GCC_VERSION=16.2.0
ARG LCOV_VERSION=2.6
ARG COMPDB_VERSION=0.2.0

FROM --platform=linux/amd64 gcc:${GCC_VERSION}-trixie AS gcc-source

FROM --platform=$BUILDPLATFORM debian:${DEBIAN_VERSION} AS fetch

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        make \
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

FROM fetch AS lcov

ARG LCOV_VERSION

RUN curl --fail --silent --show-error --location --retry 3 \
        "https://github.com/linux-test-project/lcov/releases/download/v${LCOV_VERSION}/lcov-${LCOV_VERSION}.tar.gz" \
        --output /tmp/lcov.tar.gz \
    && mkdir /tmp/lcov \
    && tar -xzf /tmp/lcov.tar.gz -C /tmp/lcov --strip-components=1 \
    && make -C /tmp/lcov install PREFIX=/opt/lcov LCOV_NO_DOC=1 LCOV_NO_XS=1 \
    && rm -rf /tmp/lcov /tmp/lcov.tar.gz /opt/lcov/share/lcov/tests

FROM --platform=linux/amd64 ${COMMON_IMAGE} AS development

ARG COMPDB_VERSION
ARG USERNAME=hydrolib

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        binutils \
        doxygen \
        gdb \
        libatomic1 \
        libbenchmark-dev \
        libc6-dev \
        libcapture-tiny-perl \
        libdatetime-perl \
        libdevel-stacktrace-perl \
        libgtest-dev \
        libjson-xs-perl \
        libtimedate-perl \
        libzstd1 \
        perl \
        python3 \
        python3-venv \
    && rm -rf /var/lib/apt/lists/*

COPY --from=gcc /opt/gcc/ /usr/local/
COPY --from=lcov /opt/lcov/ /opt/lcov/

ENV PATH="/opt/lcov/bin:/opt/compdb/bin:${PATH}" \
    CC=/usr/local/bin/gcc \
    CXX=/usr/local/bin/g++

RUN printf '%s\n' /usr/local/lib64 /usr/local/lib > /etc/ld.so.conf.d/000-gcc.conf \
    && ldconfig \
    && mkdir -p /etc/gdb/gdbinit.d \
    && printf '%s\n' \
        'add-auto-load-safe-path /usr/local/lib' \
        'add-auto-load-safe-path /usr/local/lib64' \
        > /etc/gdb/gdbinit.d/gcc.gdb \
    && ln -sf /usr/local/bin/gcc /usr/local/bin/cc \
    && ln -sf /usr/local/bin/g++ /usr/local/bin/c++ \
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

USER $USERNAME
