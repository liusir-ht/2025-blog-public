## 背景

业务上有一个新的 Java 服务，需要使用 **JDK 25**，同时服务内部需要通过 **FFmpeg** 对音视频文件进行处理。

由于 Ubuntu 24.04 默认提供的 FFmpeg 版本无法满足业务对 **FFmpeg 8.0** 的要求，因此这里采用 Ubuntu 24.04 作为基础镜像，安装 JDK 25，并通过源码编译的方式安装 FFmpeg 8.0。

最终将 JDK 25 和 FFmpeg 8.0 集成到同一个 Docker 镜像中，方便后续 Java 服务直接使用。

## 环境介绍

| 组件名称 | 组件版本 | 组件介绍 |
| --- | --- | --- |
| Ubuntu | 24.04 | Docker 基础操作系统 |
| OpenJDK | 25 | Java 运行及编译环境 |
| FFmpeg | 8.0 | 音视频处理工具 |
| GCC | Ubuntu 默认版本 | FFmpeg 源码编译工具 |
| Docker | 20.10+ | 容器运行环境 |

## 核心步骤

### 1. 使用 Ubuntu 24.04 作为基础镜像

首先使用 Ubuntu 24.04 作为 Docker 基础镜像：

```dockerfile
FROM ubuntu:24.04
```

为了避免在安装软件包过程中出现交互式配置提示，设置：

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
```

### 2. 安装 JDK 25

更新软件源并安装 OpenJDK 25，同时安装后续编译 FFmpeg 所需要的一些基础工具：

```dockerfile
RUN apt-get update && \
    apt-get install -y \
        openjdk-25-jdk \
        wget \
        git \
        build-essential \
        nasm \
        yasm \
        pkg-config && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

安装完成后设置 `JAVA_HOME`：

```dockerfile
ENV JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64
ENV PATH="$JAVA_HOME/bin:$PATH"
```

可以通过下面的命令确认 Java 版本：

```bash
java -version
```

正常情况下可以看到 JDK 25 相关版本信息。

### 3. 安装 FFmpeg 8.0 编译依赖

FFmpeg 本身支持大量音视频编码格式，如果需要使用 `libx264`、`libx265`、`libvpx`、`libopus` 等功能，需要提前安装对应的开发库。

```dockerfile
RUN apt-get update && \
    apt-get install -y \
        libx264-dev \
        libx265-dev \
        libnuma-dev \
        libvpx-dev \
        libmp3lame-dev \
        libopus-dev \
        libvorbis-dev \
        libtheora-dev \
        libwebp-dev \
        libxvidcore-dev \
        libopenjp2-7-dev \
        libopencore-amrnb-dev \
        libopencore-amrwb-dev \
        libass-dev \
        libfreetype6-dev \
        libfontconfig1-dev \
        libfribidi-dev \
        libzimg-dev \
        libvidstab-dev \
        libaom-dev \
        libssl-dev \
        libsrt-openssl-dev \
        libzmq3-dev \
        libva-dev \
        libdrm-dev \
        ocl-icd-opencl-dev \
        opencl-headers \
        libvulkan-dev \
        glslang-dev \
        libshaderc-dev \
        spirv-tools \
        frei0r-plugins-dev \
        ladspa-sdk \
        lv2-dev \
        liblilv-dev \
        libbs2b-dev \
        librubberband-dev \
        libmysofa-dev \
        flite1-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

这里没有安装 `libplacebo-dev`，主要是为了避免系统仓库中的 libplacebo 版本与 FFmpeg 8.0 编译时出现 API 兼容性问题。

如果业务实际上用不到这些额外编码器，也可以根据实际需求减少依赖，从而降低最终 Docker 镜像大小。

### 4. 下载 FFmpeg 8.0 源码

进入临时目录：

```dockerfile
WORKDIR /tmp
```

从 FFmpeg 官方地址下载 FFmpeg 8.0：

```dockerfile
RUN wget https://ffmpeg.org/releases/ffmpeg-8.0.tar.gz
```

然后解压：

```bash
tar -xzf ffmpeg-8.0.tar.gz
```

源码目录为：

```text
ffmpeg-8.0
```

### 5. 编译 FFmpeg 8.0

进入 FFmpeg 源码目录：

```bash
cd ffmpeg-8.0
```

执行配置：

```bash
./configure \
    --prefix=/usr/local \
    --enable-gpl \
    --enable-nonfree \
    --enable-libx264 \
    --enable-libx265 \
    --enable-libvpx \
    --enable-libmp3lame \
    --enable-libopus \
    --enable-libvorbis \
    --enable-libass \
    --enable-libfreetype \
    --enable-libvulkan
```

其中：

- `--prefix=/usr/local`：将 FFmpeg 安装到 `/usr/local`
- `--enable-gpl`：启用 GPL 相关组件
- `--enable-nonfree`：允许启用部分非自由组件
- `--enable-libx264`：启用 H.264 编码支持
- `--enable-libx265`：启用 H.265/HEVC 编码支持
- `--enable-libvpx`：启用 VP8/VP9 编解码支持
- `--enable-libmp3lame`：启用 MP3 编码支持
- `--enable-libopus`：启用 Opus 编解码支持
- `--enable-libvorbis`：启用 Vorbis 编解码支持
- `--enable-libass`：启用 ASS/SSA 字幕支持
- `--enable-libfreetype`：启用字体渲染支持
- `--enable-libvulkan`：启用 Vulkan 相关功能

配置完成以后开始编译：

```bash
make -j$(nproc)
```

`$(nproc)` 会自动获取当前系统 CPU 核心数量，从而使用多核并行编译。

编译完成后安装：

```bash
make install
```

最后删除源码和压缩包：

```bash
cd /
rm -rf /tmp/ffmpeg-8.0*
```

### 6. 验证 JDK 25

执行：

```bash
java -version
```

确认输出的 Java 主版本为：

```text
25
```

同时可以检查 Java 路径：

```bash
which java
```

### 7. 验证 FFmpeg 8.0

执行：

```bash
ffmpeg -version
```

正常情况下可以看到类似：

```text
ffmpeg version 8.0
```

同时可以查看实际编译参数：

```bash
ffmpeg -version | head
```

例如：

```text
configuration: --prefix=/usr/local --enable-gpl ...
```

还可以验证 `ffprobe`：

```bash
ffprobe -version
```

如果 `ffmpeg` 和 `ffprobe` 都能够正常执行，则说明安装成功。

## 完整 Dockerfile

最终 Dockerfile 如下：

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# 安装 JDK 25 和基础编译工具
RUN apt-get update && \
    apt-get install -y \
        openjdk-25-jdk \
        wget \
        git \
        build-essential \
        nasm \
        yasm \
        pkg-config && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64
ENV PATH="$JAVA_HOME/bin:$PATH"

# 安装 FFmpeg 编译依赖
RUN apt-get update && \
    apt-get install -y \
        libx264-dev \
        libx265-dev \
        libnuma-dev \
        libvpx-dev \
        libmp3lame-dev \
        libopus-dev \
        libvorbis-dev \
        libtheora-dev \
        libwebp-dev \
        libxvidcore-dev \
        libopenjp2-7-dev \
        libopencore-amrnb-dev \
        libopencore-amrwb-dev \
        libass-dev \
        libfreetype6-dev \
        libfontconfig1-dev \
        libfribidi-dev \
        libzimg-dev \
        libvidstab-dev \
        libaom-dev \
        libssl-dev \
        libsrt-openssl-dev \
        libzmq3-dev \
        libva-dev \
        libdrm-dev \
        ocl-icd-opencl-dev \
        opencl-headers \
        libvulkan-dev \
        glslang-dev \
        libshaderc-dev \
        spirv-tools \
        frei0r-plugins-dev \
        ladspa-sdk \
        lv2-dev \
        liblilv-dev \
        libbs2b-dev \
        librubberband-dev \
        libmysofa-dev \
        flite1-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /tmp

# 编译安装 FFmpeg 8.0
RUN wget https://ffmpeg.org/releases/ffmpeg-8.0.tar.gz && \
    tar -xzf ffmpeg-8.0.tar.gz && \
    cd ffmpeg-8.0 && \
    ./configure \
        --prefix=/usr/local \
        --enable-gpl \
        --enable-nonfree \
        --enable-libx264 \
        --enable-libx265 \
        --enable-libvpx \
        --enable-libmp3lame \
        --enable-libopus \
        --enable-libvorbis \
        --enable-libass \
        --enable-libfreetype \
        --enable-libvulkan && \
    make -j$(nproc) && \
    make install && \
    cd / && \
    rm -rf /tmp/ffmpeg-8.0*

ENV DEBIAN_FRONTEND=

# 验证安装
RUN java -version && \
    ffmpeg -version && \
    ffprobe -version

WORKDIR /app

CMD ["/bin/bash"]
```

## 构建镜像

Dockerfile 准备完成以后执行：

```bash
docker build -t jdk25-ffmpeg8:ubuntu24.04 .
```

构建完成后查看镜像：

```bash
docker images | grep jdk25-ffmpeg8
```

启动容器：

```bash
docker run --rm -it jdk25-ffmpeg8:ubuntu24.04
```

进入容器后分别检查：

```bash
java -version
```

以及：

```bash
ffmpeg -version
```

确认 JDK 25 和 FFmpeg 8.0 均正常即可。

## 总结

通过 Ubuntu 24.04 作为基础镜像，可以将 JDK 25 和 FFmpeg 8.0 集成到同一个 Docker 运行环境中。

JDK 25 直接通过系统软件源进行安装，FFmpeg 8.0 则通过源码编译，并根据业务需要开启 `libx264`、`libx265`、`libvpx`、`libopus`、Vulkan 等功能。

这种方式比较适合 Java 服务本身存在音视频处理需求的场景，后续 Java 应用可以直接通过 `ProcessBuilder` 等方式调用 `ffmpeg` 或 `ffprobe` 命令，而不需要单独维护 FFmpeg 服务。