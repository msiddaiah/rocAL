# ##############################################################################
# Copyright (c) 2026 Advanced Micro Devices, Inc.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
# ##############################################################################
# docker build -t rocal:compute-rocm-dkms-no-npi-hipclang-17013-ubuntu-22.04 --build-arg GPU_TARGETS=gfx942 
# --build-arg BASE_DOCKER_IMAGE=registry-sc-harbor.amd.com/rocm-ci-images/compute-rocm-dkms-no-npi-hipclang:17013-ubuntu-22.04 
# -f docker/Dockerfile .

ARG BASE_DOCKER_IMAGE=registry-sc-harbor.amd.com/rocm-ci-images/compute-rocm-dkms-no-npi-hipclang:17013-ubuntu-22.04
ARG GPU_TARGETS=gfx942
FROM ${BASE_DOCKER_IMAGE}

ENV ROCM_PATH=/opt/rocm
ENV DEBIAN_FRONTEND=noninteractive

# Install build and rocAL dependency packages (match rocAL-setup.py; MIVisionX built from source below)
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    git \
    cmake \
    wget \
    unzip \
    pkg-config \
    nasm \
    yasm \
    liblmdb-dev \
    libsndfile1-dev \
    python3-dev \
    python3-pip \
    python3-venv \
    python3-protobuf \
    libprotobuf-dev \
    libprotoc-dev \
    protobuf-compiler \
    libturbojpeg0-dev \
    libopencv-dev \
    half \
    hip-dev \
    rocjpeg-dev \
    rocdecode-dev \
    && rm -rf /var/lib/apt/lists/*

# Install libtar - https://repo.or.cz/libtar.git ; version - v1.2.20
RUN git clone -b v1.2.20 https://repo.or.cz/libtar.git && cd libtar && \
    autoreconf --force --install && CFLAGS="-fPIC" ./configure && make -j$(nproc) && sudo make install && cd ../../ 

# Use Tencent RapidJSON (v1.1.0); install headers only. Patch document.h to fix const assignment (rocm/amdclang).
RUN git clone https://github.com/Tencent/rapidjson.git  \
    && cd rapidjson && mkdir build && cd build && \
    cmake ../ && make -j$(nproc) && sudo make install && cd ../../ 

# Install PyBind11 - https://github.com/pybind/pybind11 ; version - v2.11.1
RUN git clone -b v2.11.1 https://github.com/pybind/pybind11 && cd pybind11 && mkdir build && cd build && \
    cmake -DDOWNLOAD_CATCH=ON -DDOWNLOAD_EIGEN=ON ../ && make -j4 && sudo make install && cd ../../ 

# Install RPP
RUN git clone -b develop https://github.com/msiddaiah/rpp.git && cd rpp && \
    git submodule update --init --recursive && \
    mkdir build && cd build && \
    cmake $( [ -n "$GPU_TARGETS" ] && echo "-DGPU_TARGETS=$GPU_TARGETS" ) -DBACKEND=HIP ../ && make -j$(nproc) install && cd ../../

# install MIVisionX
RUN git clone -b develop https://github.com/msiddaiah/MIVisionX.git && cd MIVisionX && \
    git submodule update --init --recursive && \
    mkdir build && cd build && cmake $( [ -n "$GPU_TARGETS" ] && echo "-DGPU_TARGETS=$GPU_TARGETS" ) -DBACKEND=HIP ../ && make -j$(nproc) && make install && cd ../../

    
WORKDIR /rocal
COPY . .

# Initialize rocAL third-party submodules
RUN git submodule update --init --recursive

# Build rocAL from the checked-out source in /rocal
RUN pip install --upgrade pip
RUN mkdir -p build && cd build && \
    cmake $( [ -n "$GPU_TARGETS" ] && echo "-DGPU_TARGETS=$GPU_TARGETS" ) \
        -DBACKEND=HIP \
    .. && \
    make -j$(nproc) && make install
ENV PYTHONPATH="/opt/rocm/lib:${PYTHONPATH}"

WORKDIR /rocal/build

