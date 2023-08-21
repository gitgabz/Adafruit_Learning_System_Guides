# syntax=docker/dockerfile:1.3-labs
# Converted the dockerfile to Buildkit so we can use heredocs to easily keep build commands and local scripts inside the dockerfile.
# Build with "DOCKER_BUILDKIT=1 docker build -t esp32buildenv:latest ."
# Build with "DOCKER_BUILDKIT=1 docker build --target esp32s3-linux-builder-toolchain -t esp32buildenv:latest ."

# Dockerfile port of https://gist.github.com/jcmvbkbc/316e6da728021c8ff670a24e674a35e6
# wifi details http://wiki.osll.ru/doku.php/etc:users:jcmvbkbc:linux-xtensa:esp32s3wifi

###############################
# Build tools base layer      #
###############################
# we need python 3.10 not 3.11
FROM ubuntu:22.04 as esp32s3-linux-builder-base

WORKDIR /app

RUN apt-get update
RUN apt-get -y install gperf bison flex texinfo help2man gawk libtool-bin git unzip ncurses-dev rsync zlib1g zlib1g-dev xz-utils cmake wget bzip2 g++ python3 python3-dev python3-pip cpio bc virtualenv libusb-1.0 && \
    ln -s /usr/bin/python3 /usr/bin/python

# install autoconf 2.71
RUN wget https://ftp.gnu.org/gnu/autoconf/autoconf-2.71.tar.xz && \
    tar -xf autoconf-2.71.tar.xz && \
    cd autoconf-2.71 && \
    ./configure --prefix=`pwd`/root && \
    make && \
    make install
ENV PATH="$PATH:/app/autoconf-2.71/root/bin"

# dynconfig
RUN git clone https://github.com/jcmvbkbc/xtensa-dynconfig -b original --depth=1 && \
    git clone https://github.com/jcmvbkbc/config-esp32s3 esp32s3 --depth=1 && \
    make -C xtensa-dynconfig ORIG=1 CONF_DIR=`pwd` esp32s3.so
ENV XTENSA_GNU_CONFIG="/app/xtensa-dynconfig/esp32s3.so"

# ct-ng cannot run as root, we'll just do everything else as a user
RUN useradd -d /app/build -u 3232 esp32 && mkdir build && chown esp32:esp32 build && chown -R esp32:esp32 /app
USER esp32

###############################
# Toolchain layer             #
###############################
FROM esp32s3-linux-builder-base as esp32s3-linux-builder-toolchain

# Add docker cache-layer check for git repository changes.
ADD ["https://api.github.com/repos/jcmvbkbc/crosstool-NG/git/refs/heads/xtensa-fdpic", \
    "https://api.github.com/repos/jcmvbkbc/buildroot/git/refs/heads/xtensa-2023.02-fdpic", \
    "https://api.github.com/repos/jcmvbkbc/esp-hosted/git/refs/heads/shmem",\
    "/tmp/git-repo-refs/"]

# toolchain
COPY --chown=esp32:esp32 <<EOF build-crosstool.sh
#!/bin/bash
  cd build && \
  git clone https://github.com/jcmvbkbc/crosstool-NG.git -b xtensa-fdpic --depth=1 && \
  cd crosstool-NG && \
  ./bootstrap && \
  ./configure --enable-local && \
  make && \
  ./ct-ng xtensa-esp32s3-linux-uclibcfdpic && \
  CT_PREFIX=`pwd`/builds ./ct-ng build & exit 0
EOF

# kernel and rootfs
COPY --chown=esp32:esp32 <<EOF build-buildroot.sh
#!/bin/bash
  cd build && \
  git clone https://github.com/jcmvbkbc/buildroot -b xtensa-2023.02-fdpic --depth=1 && \
  make -C buildroot O=`pwd`/build-xtensa-2023.02-fdpic-esp32s3 esp32s3wifi_defconfig && \
  buildroot/utils/config --file build-xtensa-2023.02-fdpic-esp32s3/.config --set-str TOOLCHAIN_EXTERNAL_PATH `pwd`/crosstool-NG/builds/xtensa-esp32s3-linux-uclibcfdpic && \
  buildroot/utils/config --file build-xtensa-2023.02-fdpic-esp32s3/.config --set-str TOOLCHAIN_EXTERNAL_PREFIX \'$(ARCH)-esp32s3-linux-uclibcfdpic\' && \
  buildroot/utils/config --file build-xtensa-2023.02-fdpic-esp32s3/.config --set-str TOOLCHAIN_EXTERNAL_CUSTOM_PREFIX \'$(ARCH)-esp32s3-linux-uclibcfdpic\' && \
  make -C buildroot O=`pwd`/build-xtensa-2023.02-fdpic-esp32s3 -f build/build-xtensa-2023.02-fdpic-esp32s3/images/xipImage -a -f build/build-xtensa-2023.02-fdpic-esp32s3/images/rootfs.cramfs || exit 1
EOF

# bootloader
ENV IDF_PATH="/app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/esp-idf"
COPY --chown=esp32:esp32 <<EOF build-esp-hosted.sh
#!/bin/bash
  cd build && \
  git clone https://github.com/jcmvbkbc/esp-hosted -b shmem --depth=1 && \
  cd esp-hosted/esp_hosted_ng/esp/esp_driver && cmake . && \
  cd esp-idf && . ./export.sh && \
  cd ../network_adapter && idf.py set-target esp32s3 && \
  cp sdkconfig.defaults.esp32s3 sdkconfig && idf.py build
EOF

# create easy entrypoint to rebuild artifacts
COPY --chown=esp32:esp32 <<EOF build-all-clean.sh
#!/bin/bash
OUTPUT=/app/build/release-local-"$(date +"%Y_%m_%d_%I_%M_%p")"
rm -rf /app/build/*
./build-crosstool.sh && ./build-buildroot.sh && ./build-esp-hosted.sh &&\
mkdir -p \$OUTPUT &&\
cp /app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/bootloader/bootloader.bin \$OUTPUT &&\
cp /app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/partition_table/partition-table.bin \$OUTPUT &&\
cp /app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/network_adapter.bin \$OUTPUT &&\
cp /app/build/build-xtensa-2023.02-fdpic-esp32s3/images/xipImage \$OUTPUT &&\
cp /app/build/build-xtensa-2023.02-fdpic-esp32s3/images/rootfs.cramfs \$OUTPUT &&\
echo "Build artifacts are copied to folder: \$OUTPUT"
EOF

USER root
RUN chmod u+x build-*.sh

USER esp32
RUN ./build-crosstool.sh && ./build-crosstool.sh && ./build-esp-hosted.sh


###############################
# Artifact layer              #
###############################
# Leaves us with a clean small build base and artifacts
# To force a new build of all layers run: 'docker build --no-cache .'
FROM esp32s3-linux-builder-base as esp32s3-linux-builder-artifacts

# move files over
COPY --from=esp32s3-linux-builder-toolchain [\
    "/app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/bootloader/bootloader.bin", \
    "/app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/partition_table/partition-table.bin", \ 
    "/app/build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/network_adapter.bin", \
    "/app/build/build-xtensa-2023.02-fdpic-esp32s3/images/xipImage", \
    "/app/build/build-xtensa-2023.02-fdpic-esp32s3/images/rootfs.cramfs", \
    "/app/build/release/"] 

# keep docker running so we can debug/rebuild :)
USER root
ENTRYPOINT ["tail", "-f", "/dev/null"]

# grab the files with `docker cp CONTAINER_NAME:/app/build/release/\* .`
# now you can burn the files from the 'release' folder with: 
# python esptool.py --chip esp32s3 -p /dev/ttyUSB0 -b 921600 --before=default_reset --after=hard_reset write_flash 0x0 bootloader.bin 0x10000 network_adapter.bin 0x8000 partition-table.bin
# next we can burn in the kernel and filesys with parttool, which is part of esp-idf
# parttool.py write_partition --partition-name linux --input xipImage
# parttool.py write_partition --partition-name rootfs --input rootfs.cramfs
