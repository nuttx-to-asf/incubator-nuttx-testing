# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.  The
# ASF licenses this file to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance with the
# License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the
# License for the specific language governing permissions and limitations
# under the License.

FROM ubuntu:19.10 AS builder-base
# NOTE WE ARE NOT REMOVEING APT CACHE.
# This should only be used for temp build images that artifacts will be copied from
RUN apt-get update -qq && apt-get install -y -qq \
  curl \
  xz-utils

###############################################################################
# Base image that should be used to prepare tools from nuttx-tools
###############################################################################
FROM builder-base AS nuttx-tools

RUN apt-get install -y -qq \
  flex \
  bison \
  gperf \
  libncurses5-dev \
  make

RUN mkdir /tools
WORKDIR /tools

RUN mkdir -p /tools/nuttx-tools
RUN curl -s -L https://bitbucket.org/nuttx/tools/get/9ad3e1ee75c7.tar.gz \
  | tar -C nuttx-tools --strip-components=1 -xz

RUN cd nuttx-tools/kconfig-frontends \
  && ./configure --enable-mconf --disable-gconf --disable-qconf --enable-static --prefix=/tools/kconfig-frontends \
  && make install

RUN cd nuttx-tools \
  && mkdir genromfs \
  && tar -C genromfs --strip-components=1 -xf genromfs-0.5.2.tar.gz \
  && cd genromfs \
  && make install PREFIX=/tools/genromfs

CMD [ "/bin/bash" ]

###############################################################################
# Base image that should be used to prepare arch build images
###############################################################################
FROM builder-base AS nuttx-toolchain-base

RUN mkdir /tools
WORKDIR /tools

###############################################################################
# Build image for tool required by ARM builds
###############################################################################
FROM nuttx-toolchain-base AS nuttx-toolchain-arm
# Download the latest ARM GCC toolchain prebuilt by ARM
RUN mkdir gcc-arm-none-eabi && \
  curl -s -L  "https://developer.arm.com/-/media/Files/downloads/gnu-rm/9-2019q4/gcc-arm-none-eabi-9-2019-q4-major-x86_64-linux.tar.bz2?revision=108bd959-44bd-4619-9c19-26187abf5225&la=en&hash=E788CE92E5DFD64B2A8C246BBA91A249CB8E2D2D" \
  | tar -C gcc-arm-none-eabi --strip-components 1 -xj

###############################################################################
# Build image for tool required by Pinguino builds
###############################################################################
FROM nuttx-toolchain-base AS nuttx-toolchain-pinguino
# Download the pinguino compilers. Note this includes both 8bit and 32bit
# toolchains and builds for multiple host systems. Only copy what is needed.
RUN mkdir pinguino-compilers && \
  curl -s -L "https://github.com/PinguinoIDE/pinguino-compilers/archive/62db5158d7f6d41c6fadb05de81cc31dd81a1958.tar.gz" \
  | tar -C pinguino-compilers --strip-components=2 --wildcards -xz */linux64

###############################################################################
# Build image for tool required by RISCV builds
###############################################################################
FROM nuttx-toolchain-base AS nuttx-toolchain-riscv
# Download the latest RISCV GCC toolchain prebuilt by SiFive
RUN mkdir riscv64-unknown-elf-gcc && \
  curl -s -L "https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14.tar.gz" \
  | tar -C riscv64-unknown-elf-gcc --strip-components 1 -xz

###############################################################################
# Build image for tool required by ESP32 builds
###############################################################################
FROM nuttx-toolchain-base AS nuttx-toolchain-esp32
# Download the latest ESP32 GCC toolchain prebuilt by Espressif
RUN mkdir xtensa-esp32-elf-gcc && \
  curl -s -L "https://dl.espressif.com/dl/xtensa-esp32-elf-gcc8_2_0-esp32-2019r1-rc2-linux-amd64.tar.xz" \
  | tar -C xtensa-esp32-elf-gcc --strip-components 1 -xJ

RUN apt-get install -y -qq --no-install-recommends \
  git bison gperf python python-pip python-setuptools make cmake ninja-build ccache libffi-dev libssl-dev libusb-1.0
RUN git clone --depth 1 --shallow-submodules --recursive https://github.com/espressif/esp-idf.git
# This is unfortunatly going to re-download some of the same toolchains, but will only be used in the context of esp-idf
RUN bash -c "\
  cd esp-idf && \
  ./install.sh"

# Provide an option for disabling DOUT mode, so that the default one,
# DIO, can be used (for instance, for flashing a DevKitC module)
ARG FLASH_DOUT_ENABLED=y
# We run this is generate the default bootloader and partition table binaries
RUN bash -c "\
  cd esp-idf && \
  source ./export.sh && \
  cd examples/get-started/hello_world && \
  make defconfig && \
  if [[ '${FLASH_DOUT_ENABLED}' == y ]]; then echo 'CONFIG_ESPTOOLPY_FLASHMODE_DOUT=y' >> sdkconfig; fi && \
  idf.py bootloader partition_table"

RUN mkdir /tools/blobs
RUN cp /tools/esp-idf/examples/get-started/hello_world/build/bootloader/bootloader.bin /tools/blobs/
RUN cp /tools/esp-idf/examples/get-started/hello_world/build/partition_table/partition-table.bin /tools/blobs/


###############################################################################
# Final Docker image used for running CI system.  This includes all toolchains
# supported by the CI system.
###############################################################################
FROM ubuntu:19.10
LABEL maintainer="dev@nuttx.apache.org"

RUN dpkg --add-architecture i386
# This is used for the final images so make sure to not store apt cache
RUN apt-get update -qq && apt-get install -y -qq --no-install-recommends \
  git \
  build-essential \
  gcc \
  gcc-multilib \
  gcc-avr \
  avr-libc \
  wget \
  libx11-dev \
  libxext-dev \
  u-boot-tools \
  lib32z1-dev \
  libc6-dev-i386 \
  libx11-dev:i386 \
  libxext-dev:i386 \
  linux-libc-dev:i386 \
  curl \
  gettext \
  libcurl4-openssl-dev \
  xxd \
  unzip \
  python3 \
  python3-pip \
  && rm -rf /var/lib/apt/lists/*


# Configure out base setup for adding python packages
ENV PIP_DISABLE_PIP_VERSION_CHECK=true
# This disables the cache with value 0. We do not want caching as it
# increases the images size.
ENV PIP_NO_CACHE_DIR=0
# We are using the minimal python installation from the system so include
# setuptools and also wheel so we can use the binary releases of packages
# instead of requiring them to be compiled.
RUN pip3 install setuptools wheel

RUN mkdir /tools
WORKDIR /tools

# Pull in the tools we just built for nuttx
COPY --from=nuttx-tools /tools/genromfs/ /tools/genromfs/
ENV PATH="/tools/genromfs/usr/bin:$PATH"
COPY --from=nuttx-tools /tools/kconfig-frontends/ kconfig-frontends/
ENV PATH="/tools/kconfig-frontends/bin:$PATH"

# ARM toolchain
COPY --from=nuttx-toolchain-arm /tools/gcc-arm-none-eabi/ gcc-arm-none-eabi/
ENV PATH="/tools/gcc-arm-none-eabi/bin:$PATH"

# MIPS toolchain
COPY --from=nuttx-toolchain-pinguino /tools/pinguino-compilers/p32/ pinguino-compilers/p32/
ENV PATH="/tools/pinguino-compilers/p32/bin:$PATH"

# RISCV toolchain
COPY --from=nuttx-toolchain-riscv /tools/riscv64-unknown-elf-gcc/ riscv64-unknown-elf-gcc/
ENV PATH="/tools/riscv64-unknown-elf-gcc/bin:$PATH"

# ESP32 toolchain
COPY --from=nuttx-toolchain-esp32 /tools/xtensa-esp32-elf-gcc/ xtensa-esp32-elf-gcc/
COPY --from=nuttx-toolchain-esp32 /tools/esp-idf/ esp-idf/
ENV PATH="/tools/xtensa-esp32-elf-gcc/bin:$PATH"
RUN mkdir -p /tools/blobs/esp32core
COPY --from=nuttx-toolchain-esp32 /tools/blobs/* /tools/blobs/esp32core/
RUN pip3 install esptool

CMD [ "/bin/bash" ]
