---
title: "用 chroot 突破安卓 32 位限制,跑起 64 位程序"
date: 2026-07-01 09:00:00 +0800
categories: [AI, Sth, LLM]
tags: [ AI, chroot , llama-cpp]
---

一台32位安卓设备要跑 **64 位**的程序(下文以本地大模型 llama.cpp 作为验证案例)。用户空间缺乏 64 位运行时,看似不可行,而 `chroot` 恰好能补上这块缺口。这篇文章用一次真实部署,讲清楚 chroot 到底干了什么、以及为什么它在这台设备上恰好是最优解。

> 文中命令以一台真实的安卓设备为例(全志 H618,4×Cortex-A53;用户空间 32 位 / 内核 64 位)
{: .prompt-info }

## 用户空间与内核位宽不一致

先看几个探测结果:

| 看到的 | 真相 |
| --- | --- |
| `getprop ro.product.cpu.abilist` → `armeabi-v7a,armeabi` | 安卓用户空间是 **32 位** |
| `ls /system/bin/linker64` → 不存在 | 系统里**没有 64 位库** |
| `uname -a` → `... armv8l` | 但**内核是 64 位**(`armv8l` = ARMv8 运行在 32 位视角) |

以 llama.cpp 这类本地大模型工具来验证(如今基本只发布 arm64 版本)。直接跑会报 `No such file or directory`——因为找不到 64 位动态链接器 `ld-linux-aarch64.so`。

既然内核本身是 64 位,这条路径就成立——关键在于 **chroot**。

## chroot:切换进程的文件系统根

**chroot = change root,「换根」。**

正常情况下,进程看到的根目录 `/` 是系统根(里面有 `/system`、`/data`、`/proc`…)。`chroot <目录> <命令>` 做的事就是:**让这个命令把 `<目录>` 当成自己的 `/`**。

于是在这个命令眼里,文件系统从那个目录开始,它**看不到外面**的世界,只能看到你给它准备的那棵目录树。

换言之,chroot 改变的是进程对文件系统的**视图起点**,而非底层运行环境:内核与 CPU 均不变,但进程可见的程序、库、配置被整体替换为另一套。

## 关键:内核 ≠ 用户空间

这是理解「为什么 chroot 能绕过 32 位限制」的核心。

一个程序是 32 位还是 64 位,**由它自身的 ELF 文件决定,跟安卓用户空间无关**;真正执行程序的,是**内核**。

- 设备内核是 **64 位 aarch64**;
- 64 位内核**原生就能执行 64 位 ELF**(本职工作),开启 `CONFIG_COMPAT` 后也能执行 32 位 ELF;
- 安卓之所以「是 32 位」,只是因为它**装的全是 32 位程序 + 32 位 Bionic 库**。内核本身具备执行 64 位 ELF 的能力,只是用户空间未提供相应的程序与库。

> **「32 位安卓」是用户空间的属性,不是内核的属性。** 只要给内核准备一套 64 位的运行时(程序 + libc + 库),它就能跑。chroot 干的就是这件事——将一套完整的 64 位 Linux 运行时(Alpine aarch64 rootfs)挂载到某个目录,随后让进程在其中运行。
{: .prompt-tip }

## 第一步:最小验证

先用最小代价确认内核能执行 aarch64 ELF——下载 Alpine mini rootfs(~4MB),用自带的 busybox 跑通 chroot:

```bash
ROOT=/data/local/tmp/alpine
mkdir -p $ROOT/{proc,dev,sys}
# 下载并解压 aarch64 rootfs
curl -L -o /tmp/rootfs.tar.gz \
  https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.0-aarch64.tar.gz
tar xzf /tmp/rootfs.tar.gz -C $ROOT
# 三条命令挂载并进入
mount -t proc  none  $ROOT/proc      # ①
mount --bind /dev    $ROOT/dev        # ②
chroot $ROOT /bin/busybox echo "OK"   # ③
```

- **① `mount -t proc none $ROOT/proc`** —— chroot 后进程看不到外面的 `/proc`(根都换了)。重新挂一份 `proc`,让里面的程序能读进程、内存、CPU 信息(很多程序启动要看 `/proc/cpuinfo`、`/proc/self/…`)。
- **② `mount --bind /dev $ROOT/dev`** —— 绑定挂载:把外部真实的 `/dev`「投影」到 `$ROOT/dev`,里面的程序就能用 `/dev/null`、`/dev/urandom`、终端设备等基础节点。
- **③ `chroot $ROOT /bin/busybox echo "OK"`** —— 把 `$ROOT` 设为新根,在新根里找 `/bin/busybox` 执行。这个 busybox 是 **aarch64 64 位**二进制(由 Alpine rootfs 提供),由内核直接执行 → 输出 `OK`。

看到 `OK` 即证明:**64 位内核能执行 aarch64 二进制**,方案成立。这一步只验证可行性,环境本身还不具备跑真实程序的能力——配置是下一步的事。

## 第二步:配置 chroot 环境

最小验证只用到 rootfs 自带的 busybox,要让它成为可用的 64 位环境,先配置 DNS 与包管理源(使 `apk` 能联网装包):

```bash
printf "nameserver 223.5.5.5\nnameserver 114.114.114.114\n" > $ROOT/etc/resolv.conf
printf "http://mirrors.tuna.tsinghua.edu.cn/alpine/v3.20/main\nhttp://mirrors.tuna.tsinghua.edu.cn/alpine/v3.20/community\n" > $ROOT/etc/apk/repositories
```

随后用统一模板进入 chroot,后续操作都在其内进行:

```bash
chroot $ROOT /usr/bin/env -i PATH=/usr/sbin:/usr/bin:/sbin:/bin HOME=/root /bin/sh
```

> `PATH` 必须包含 `/sbin`——`apk` 正位于该目录,遗漏会报 `apk: not found`。
{: .prompt-info }

## 第三步:安装编译工具链

进入 chroot 后,先装编译所需的工具链:

```bash
apk update
apk add --no-progress build-base cmake git wget linux-headers
```

> ⚠️ **`linux-headers` 不能漏。** Alpine 的 `build-base` 不含内核头文件;若漏装,编译到 `common/arg.cpp` 会因找不到 `linux/limits.h` 而中断。
{: .prompt-warning }

## 第四步:编译 llama.cpp

工具链就绪,开始编译 llama.cpp(它只发布 arm64,正好用来验证这套 64 位运行时):

```bash
git clone --depth 1 https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release \
  -DGGML_NATIVE=OFF -DGGML_ARM_ARCH=armv8-a
cmake --build build -j4 --target llama-cli llama-server
```

> ⚠️ **关键坑:`GGML_NATIVE=OFF` 不能省。** Cortex-A53 上 native 检测会给代码加 `+nofp` target attribute,与浮点代码冲突,触发 gcc `internal compiler error`。改用显式 `armv8-a` baseline 还附带一个好处——产物可在所有 ARMv8 CPU 上运行,便于跨型号分发。
{: .prompt-warning }

编译产物(`build/bin/llama-cli`、`build/bin/llama-server` 等)是原生 aarch64 二进制,链接 chroot 内的 musl libc,由宿主 64 位内核直接执行。下一步让它真正跑起来。

## 第五步:下载模型并启动

编译出的只是引擎,还需要一个量化模型才能推理。从 modelscope 拉取(国内速度远优于 HuggingFace 直连):

```bash
mkdir -p /root/models
curl -L -o /root/models/qwen0.5b-q4_k_m.gguf \
  "https://modelscope.cn/models/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/master/qwen2.5-0.5b-instruct-q4_k_m.gguf"
```

后台启动 `llama-server`,暴露 OpenAI 兼容 API:

```bash
setsid ./build2/bin/llama-server \
  -m /root/models/qwen0.5b-q4_k_m.gguf \
  --host 0.0.0.0 --port 8080 -t 4 -c 2048 \
  </dev/null >>/tmp/server.log 2>&1 &
```

待 `curl http://127.0.0.1:8080/health` 返回 `{"status":"ok"}` 即就绪,发一个对话请求验证整条链路:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen","messages":[{"role":"user","content":"用一句话介绍你自己"}],"max_tokens":60}'
```

返回标准 OpenAI 格式即代表贯通——64 位 rootfs → 编译 → 模型 → 推理服务,全程跑在 32 位安卓的 64 位内核之上,实测约 **5 tok/s**,前文所述的「裸机性能」就此落地。

## 第六步:跑个中文 TTS(edge-tts)

同一套 chroot 既能跑本地大模型,也能跑其它 arm64 生态工具。再以**中文文本转语音(TTS)**为例,看看这套环境的复用价值。

本地 TTS 模型(VITS 一类)即便走非自回归的 ONNX,在这颗 A53 上实测也要 RTF 3~6(几倍慢于实时)。**edge-tts** 走另一条路:它是微软 Edge 在线 Neural TTS 的 Python 客户端,**不在本地跑模型**,只发 HTTP 给微软云端、收回 mp3。盒子只做网络收发,A53 几乎不参与计算,因此能**即时**返回,代价是要联网。

> edge-tts 只依赖 aiohttp(网络库),不依赖 onnxruntime,因此能直接装进这套 Alpine musl 环境,无需再为 glibc 另建 chroot。
{: .prompt-info }

仍在 chroot 内,装 Python 与 edge-tts:

```bash
apk add --no-progress python3 py3-pip
pip3 install --break-system-packages edge-tts \
  -i https://pypi.tuna.tsinghua.edu.cn/simple
```

> ⚠️ **`externally-managed` 报错。** Alpine 的 Python 3.12 启用了 PEP 668,pip 默认拒绝改动系统包。加 `--break-system-packages` 绕过(单机测试可接受);正式做法是建 venv。
{: .prompt-warning }

合成一句中文。文本带中文标点时,写进文件用 `--file` 传入,避开 shell 多层引号转义:

```bash
echo "你好,这是一段中文语音合成测试。" > /root/text.txt
edge-tts --voice zh-CN-XiaoxiaoNeural \
  --file /root/text.txt --write-media /root/out.mp3
```

常用中文音色:`zh-CN-XiaoxiaoNeural`(女)、`zh-CN-YunxiNeural`(男)、`zh-CN-YunyangNeural`(男新闻);`edge-tts --list-voices | grep zh-CN` 列全部。默认输出 mp3(24kHz / 48kbps mono)。

实测一段约 14 秒的音频,**3.5 秒**生成完——约 4 倍快于实时,CPU 几乎闲置。重活在微软云端,A53 只做网络收发。把 mp3 拉回开发机即可试听:

```bash
adb pull /data/local/tmp/alpine/root/out.mp3 ./out.mp3
```

同一套 chroot:llama.cpp 在本地慢条斯理地推理,edge-tts 在线即时出语音——环境只搭一次,能跑的东西却很多。

## chroot 与虚拟机、容器的区别

容易把它和虚拟机、容器混淆,其实差别很大:

|  | chroot | 虚拟机(VM) | 容器(Docker) |
| --- | --- | --- | --- |
| 独立内核 | ❌ 共享宿主内核 | ✅ 有自己的内核 | ❌ 共享内核 |
| 文件系统隔离 | ✅ 换个根 | ✅ 完整磁盘 | ✅ 分层文件系统 |
| 进程/PID/网络隔离 | ❌ 全共享 | ✅ 完全隔离 | ✅ namespace 隔离 |
| 资源限制(cgroup) | ❌ 无 | ✅ 有 | ✅ 有 |
| 性能开销 | **几乎为零** | 大 | 小 |

三个关键点:

1. **不是虚拟机**:没有独立内核,所有 syscall 直接由宿主内核处理,指令直接在 CPU 上执行,**零性能损失**。程序跑起来就是裸机性能(实测本地大模型 ~5 tok/s),中间没有一层「虚拟化」。
2. **不是容器**:没有 namespace,chroot 里的进程和外面**共用进程表、网络栈、PID**。`ps` 能看到外面所有进程,它绑的端口就是宿主端口。
3. **只隔离文件系统视图**:它只决定进程把哪个目录当 `/`,从而决定能看到哪些程序和库。**仅此而已。**

## 方案选型:为何 chroot 是最优解

对比其他方案在本场景下的可行性:

| 方案 | 结论 | 原因 |
| --- | --- | --- |
| **NDK 编译安卓原生二进制** | ❌ 行不通 | 产物要链接 64 位 Bionic 库,但这台设备连 `linker64` 都没有——安卓侧根本没有 64 位运行时 |
| **静态链接单文件** | ⚠️ 可行但麻烦 | 内核能直接执行静态 ELF,但每个工具都得静态编,不少库静态链接有坑 |
| **虚拟机 / Docker** | ❌ 跑不了 | 内核没 KVM、没完整 namespace/cgroup,性能也扛不住 |
| **chroot Linux** | ✅ **最优** | 自带完整 64 位运行时,内核原生执行,零开销,生态完整(直接 `apk` 装包、`git clone` 编译) |

注意一个反直觉的点:**chroot 能成,恰恰因为内核是 64 位的。** chroot 解决的是「用户空间缺 64 位运行时」——将整套运行时挂载进来。如果内核本身是 32 位的(老 ARMv7 设备),无论准备多少个 64 位 rootfs 都无济于事,内核本身无法识别 64 位 ELF。

## 运行时观测

部署中观测到以下几个现象,可印证上述原理:

1. **chroot 内 `uname` 仍报 `armv8l`** —— 内核是宿主共享的,uname 直接问内核,内核在 32 位兼容视角下报 `armv8l`。但不影响:编译用的 gcc 是 aarch64,产物 `file` 是 `ELF 64-bit`,跑起来也是 64 位。
2. **`built with GNU 13.2.1 for Linux armv8l`** —— llama.cpp 启动时打印的系统标识来自 uname;而 CPU 特性检测显示 `NEON=1, ARM_FMA=1`,二进制真实架构是 aarch64,内核照常执行。
3. **server 监听 8080,局域网直接可访问** —— 没有网络 namespace,chroot 里的 server 绑的就是宿主网络栈,外面 `curl http://设备IP:8080` 直通。
4. **5 tok/s** —— 纯裸机性能,执行路径上没有虚拟化层。

## 局限性与注意事项

- **需要 root**:`chroot`、`mount` 都是特权操作。
- **内核共享,一损俱损**:chroot 里程序的 syscall 直接进宿主内核;它没有「保护宿主」的能力,内核崩了它也崩。
- **隔离是「软」的**:root 用户历史上能逃逸 chroot(chroot escape)。可信单用户环境不担心,但别把它当安全沙箱。
- **SELinux 可能拦**:有些设备策略会拒 `mount --bind` / `chroot`,需 `setenforce 0` 或调策略。
- **不限制资源**:没有 cgroup,chroot 内的程序可耗尽宿主 CPU/内存(如编译时 4 核打满,即直接占用宿主资源)。

## 结语

> chroot 使同一台设备上可以并存另一个完整 Linux 运行时:**共享内核,隔离文件系统**。在这台 32 位安卓设备上,它将一套 64 位 Linux 运行时挂载到 `/data/local/tmp/alpine`,使 64 位内核得以直接执行 64 位程序,从而绕开安卓用户空间的 32 位限制。本地大模型、任意 arm64 工具均可借此在此类设备上运行。chroot 既非虚拟机也非容器,本质只是一次文件系统根的切换——却恰好弥合了用户空间与内核之间的位宽鸿沟。
{: .prompt-info }