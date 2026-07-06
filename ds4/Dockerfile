# Dockerfile

FROM ubuntu:26.04

# RUN dpkg -l | grep -i hipblas
# RUN find / -name "hipblas.h" 2>/dev/null

RUN groupadd -g 991 render || true

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    make \
    git \
    ca-certificates \
    hipcc \
    rocminfo \
    rocm-smi \
    libamdhip64-dev \
    libhipblas-dev \
    libhipblaslt-dev \
    librocblas-dev \
    librocwmma-dev \
    libhipcub-dev \
    && rm -rf /var/lib/apt/lists/*

# librocwmma-dev ships incomplete headers (missing rocwmma/internal/) —
# same issue as the earlier bare-metal build. Pull a complete tree.
RUN git clone --depth 1 --branch rocm-7.1.0 \
      https://github.com/ROCm/rocWMMA.git /tmp/rocWMMA-rocm-7.1.0 \
    && mkdir -p /usr/local/include \
    && cp -a /tmp/rocWMMA-rocm-7.1.0/library/include/rocwmma /usr/local/include/ \
    && rm -rf /tmp/rocWMMA-rocm-7.1.0

ARG DS4_REPO_URL=https://github.com/antirez/ds4
ARG DS4_REF=main

WORKDIR /build
RUN git clone --depth 1 --branch "${DS4_REF}" "${DS4_REPO_URL}" ds4

WORKDIR /build/ds4
RUN make rocm

WORKDIR /app
RUN cp /build/ds4/ds4 /build/ds4/ds4-server /build/ds4/ds4-bench \
       /build/ds4/ds4-eval /build/ds4/ds4-agent /app/

ENV HSA_OVERRIDE_GFX_VERSION=11.5.1

ENTRYPOINT ["./ds4-server"]
