FROM nvidia/cuda:10.1-devel-ubuntu18.04

ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,video,utility

WORKDIR /tmp

RUN apt-get update -qq && apt-get -y install \
    autoconf \
    automake \
    build-essential \
    cmake \
    libtool \
    pkg-config \
    wget \
    yasm \
    git

RUN wget -O nv-codec-headers.tar.gz https://github.com/FFmpeg/nv-codec-headers/archive/master.tar.gz && \
    mkdir nv-codec-headers && cd nv-codec-headers && tar -xzvf ../nv-codec-headers.tar.gz --strip-components=1 && rm ../nv-codec-headers.tar.gz && \
    make && make install

RUN git clone https://github.com/FFmpeg/FFmpeg.git ffmpeg

WORKDIR /tmp/ffmpeg

# n4.2-dev-1308-geeca67e is the latest version in master as of time of writing
RUN git checkout n4.2-dev-1308-geeca67e

RUN ./configure \
    --disable-ffplay \
    --disable-doc \
    --disable-devices \
    --enable-cuda \
    --enable-cuvid \
    --enable-nvenc \
    --enable-nonfree \
    --extra-cflags='-I/usr/local/cuda/include' \
    --extra-ldflags='-L/usr/local/cuda/lib64' \
    --enable-cuda-nvcc

RUN make -j$(nproc --all) && make install
