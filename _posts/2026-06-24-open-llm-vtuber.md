---
title: 部署 AI 数字人(Open-LLM-VTuber)
date: 2026-06-24 08:30:00 +0800
categories: [AI, Sth , Digital]
tags: [AI, Open-LLM-VTuber]
---

[Open-LLM-VTuber](https://github.com/Open-LLM-VTuber/Open-LLM-VTuber) 是一个开源的 Live2D AI 数字人项目:网页里站着一个会说话、有口型和表情的虚拟角色,背后把**听(ASR)、想(LLM)、说(TTS)、动(Live2D)**串成一条实时对话链路。

这篇文章记录在 **Mac(Apple Silicon / arm64)** 上用 **Docker** 部署它的完整过程。目标有三个:**不污染本地环境**(全部运行在容器内)、**LLM 走 OpenAI 兼容的聚合平台**(换模型只改一行)、以及记录过程中那些"官方文档没说清楚"的坑。

## 一、整体架构

数字人本质是几个可替换模块的编排,Open-LLM-VTuber 用工厂模式把它们解耦,各环节都能单独换实现:

| 环节 | 本文方案 | 说明 |
| --- | --- | --- |
| 后端 | Docker 容器,官方预构建镜像 | arm64、纯 CPU、约 1.5GB;提供 WebSocket + Web 前端,端口 **12393** |
| LLM(大脑) | OpenAI 兼容聚合平台 | **换模型 = 改一行 `model:`** |
| ASR(听) | sherpa-onnx SenseVoice,CPU | 模型缓存在挂载卷里 |
| TTS(说) | edge-tts(微软免费云) | 零本地算力 |
| 形象 | Live2D 模型 | 浏览器 / 桌面客户端渲染 |

> 最值得关注的是 LLM 这一层:后端只认一个 **OpenAI 兼容**的 `base_url + key + model`,所以"换大脑"无需改动任何代码,改 `conf.yaml` 里一行模型名、重启即生效。这也是把它接到聚合平台的意义——同一个数字人,可在 GPT / Claude / DeepSeek / GLM 之间快速切换。
{: .prompt-tip }

## 二、官方 Docker 文档

动手前我先翻了官方文档,上面写着镜像约 **13GB**、且**仅支持 NVIDIA GPU**——这对一台 Mac 来说几乎等同于"不可用"。

但这是**旧镜像**(`t41372/open-llm-vtuber`,带 CUDA)的描述。现在官方维护的是另一个**多架构镜像**:

```
openllmvtuber/open-llm-vtuber:latest
```

它的 `dockerfile` 是 `FROM python:3.10-slim`,**纯 CPU、含 linux/arm64**,下载约 0.63GB、落盘约 1.5GB。Mac 直接拉取即可使用,完全不需要 GPU,也无需自行构建镜像。

> 结论:**别看官方文档里"13GB / 仅 NVIDIA"的说法**,那是历史镜像。认准 `openllmvtuber/open-llm-vtuber` 这个多架构镜像即可。
{: .prompt-warning }

## 三、最小化的项目结构

因为用官方预构建镜像,**不需要克隆源码**,本地只保留运行必需的几样东西:

```
open-llm-vtuber/
├── docker-compose.yml   # 编排:拉镜像 + 挂载 + 端口
├── conf/
│   ├── conf.yaml        # 主配置(LLM / ASR / TTS / host / 形象)
│   └── model_dict.json  # Live2D 模型注册表(换形象用)
└── models/
    └── sherpa-onnx-sense-voice-.../   # ASR 模型(首启自动下载)
```

`docker-compose.yml` 也很短:

```yaml
services:
  open-llm-vtuber:
    image: openllmvtuber/open-llm-vtuber:latest
    container_name: open-llm-vtuber
    ports:
      - "12393:12393"
    volumes:
      - ./conf:/app/conf       # conf.yaml 在这里,改配置即生效
      - ./models:/app/models   # 本地模型缓存,重建容器不丢
    restart: unless-stopped
```

把 `conf/` 和 `models/` 挂出来是关键:**配置和模型留在宿主机**,容器重建也不会丢失,且能直接用编辑器修改配置。

## 四、关键配置 conf.yaml

配置文件采用「**选择器字段 + 同名配置块**」模式:用 `xxx_model` / `xxx_provider` 选引擎,再到同名块里填该引擎的参数。三处要点:

#### 1. 监听地址必须是 0.0.0.0

```yaml
system_config:
  host: '0.0.0.0'   # 默认是 localhost,会导致宿主浏览器访问不到
  port: 12393
```

> 这是个很隐蔽的坑:容器里 `host: localhost` 只绑定容器内回环,Docker 端口映射模式下宿主机 `curl` 会返回 **HTTP 000**(连不上)。必须改成 `0.0.0.0`。
{: .prompt-warning }

#### 2. LLM 指向聚合平台(换模型改这里)

```yaml
character_config:
  agent_config:
    agent_settings:
      basic_memory_agent:
        llm_provider: 'openai_compatible_llm'
    llm_configs:
      openai_compatible_llm:
        base_url: 'https://api.ipsunion.com/v1'
        llm_api_key: '<你的平台 API Key>'
        model: 'gpt-5.5'          # ← 换模型只改这一行
```

> 凡是 OpenAI 兼容的服务(各类聚合平台、自建 vLLM / Ollama 等)都能这样接:`base_url` 填到 `/v1`,`model` 填平台侧的模型 ID。想从 `gpt-5.5` 换成 `deepseek-chat` / `claude-...` / `glm-4.6`,改 `model:` 这一行、重启容器即可,数字人本身不动。
{: .prompt-tip }

#### 3. ASR / TTS

```yaml
asr_config:
  asr_model: 'sherpa_onnx_asr'    # SenseVoice,CPU,首次自动下载模型
tts_config:
  tts_model: 'edge_tts'           # 微软免费云 TTS,零本地算力
```

edge-tts 走微软云、不占本地算力,适合做 demo;想完全离线可换 `sherpa_onnx_tts` 等本地引擎。

## 五、启动与验证

```bash
docker compose up -d        # 首次拉镜像(~0.63GB)
docker compose logs -f      # 看初始化(首次会下载 ASR 模型)
```

浏览器打开 <http://localhost:12393>,就能看到 Live2D 角色,可以打字或语音对话。一条完整链路是这样的:

```
你的输入 → 容器 → 聚合平台(选定的 model) → 回复 → edge-tts 念出 → Live2D 口型
```

链路是否真打通,网页里发一句就知道。也可以先在命令行直测 LLM 端,排除平台侧问题:

```bash
curl -X POST https://api.ipsunion.com/v1/chat/completions \
  -H "Authorization: Bearer <你的平台 API Key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.5","messages":[{"role":"user","content":"hi"}],"max_tokens":16}'
```

## 六、定制:换模型 / 换音色 / 换形象 / 改人设

所有定制都是同一个套路:**改 `conf/conf.yaml` → `docker compose restart`**。

- **换模型**:改 `model:` 一行(见第四节)。同一个数字人即可更换"大脑"。
- **换音色**:改 `tts_config.edge_tts.voice`,如 `zh-CN-XiaoxiaoNeural`(晓晓)换成 `zh-CN-YunxiNeural`(云希,男声)。列出全部中文音色:

  ```bash
  docker exec open-llm-vtuber python -m edge_tts --list-voices | grep zh
  ```

- **换形象**:改 `character_config.live2d_model_name`。用自定义模型时,把模型放进 `conf/live2d-models/<名>/`,并在 `conf/model_dict.json` 注册(`name` / `url` / `kScale` / `emotionMap`),容器启动会自动软链覆盖镜像内置模型。
- **改人设**:`character_config.persona_prompt` 写角色设定。注意中文预设模板自带的是一个"尖酸刻薄"的玩笑人设,正式场景记得换成贴合用途的设定。

> 自定义 Live2D 模型若没有表情文件(expressions),口型和动作正常,但不会有情绪表情——`emotionMap` 留空即可,不影响对话。
{: .prompt-info }

## 七、前端:Web 模式 vs 桌宠模式

- **Web 模式**:已随后端一起部署,访客浏览器直接开 12393,**零安装**,做演示首选。
- **桌宠 / 窗口模式**:需要独立的 **Electron 桌面客户端**(透明悬浮桌面、鼠标穿透、多屏),**浏览器不支持**,且**无法 Docker 化**——它会在本地装一个 app。

桌面客户端从前端仓库的 [Releases](https://github.com/Open-LLM-VTuber/Open-LLM-VTuber-Web/releases) 下 `.dmg` 安装,启动后在设置里把后端指向 `ws://localhost:12393` 即可连本机 Docker 后端。

> macOS 首次打开这个 `.dmg` 应用会被 **Gatekeeper 拦截**(未签名/未公证,提示"已损坏"或"无法打开")。装到 `/Applications` 后,先移除隔离属性再打开:
>
> ```bash
> xattr -rd com.apple.quarantine /Applications/open-llm-vtuber-electron.app
> ```
{: .prompt-warning }

## 八、运维与踩坑合集

日常运维命令:

```bash
docker compose ps        # 状态
docker compose stop      # 停(日常用这个)
docker compose start     # 启
docker compose restart   # 改 conf 后重启生效
docker compose logs -f   # 日志
```

> **不要随手 `docker compose down`**:容器内的前端是首次启动时现场拉取到容器层的,`down` 删掉容器后,下次启动需要联网重拉。日常用 `stop` / `start` 即可,镜像和模型都已缓存,启动是秒级。
{: .prompt-warning }

下面几个坑在国内网络环境下尤其容易遇到,单独列出:

**① Docker registry-mirrors 坏源导致拉取失败。** `daemon.json` 里配的国内镜像源(daocloud 等)可能让 `docker pull` 报 `failed size validation` / 403。解决:**清空 `registry-mirrors`,直连官方 Docker Hub**(Docker Engine 设置里删掉那段 → Apply & Restart)。

**② 代理(如 Clash 的 TUN / fake-ip 198.18.x.x)干扰大文件下载。** apt、GitHub 大文件下载都可能因此失败。ASR 模型(SenseVoice,约 1GB)可以用加速地址手动下好放进 `models/`,容器重启会自动解压、跳过下载:

```bash
curl -L -o models/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17.tar.bz2 \
  "https://gh-proxy.com/https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17.tar.bz2"
```

**③ 改 `conf.yaml` 时编辑器反复报"文件已被修改"。** 容器运行时有个 `config_sync` 会持续规范化配置文件。正确姿势:直接改文件 → 保存 → `docker compose restart`;若编辑器冲突检查太烦,用脚本直接写文件再重启。

## 九、小结

整条路走下来,在 Mac 上跑一个 AI 数字人并不算重——**一个官方 arm64 镜像 + 一份 `conf.yaml`** 就够了,全程不向本地安装 Python,模型与运行时都封装在容器内。

而真正的灵活性在于那层 OpenAI 兼容的 LLM 解耦:数字人是固定的"外壳",大脑则可自由替换。把 `base_url` 指向任意聚合平台,改一行 `model:` 就能让同一个角色切换不同模型——这也是把它当 demo 时最直观的一个卖点。
