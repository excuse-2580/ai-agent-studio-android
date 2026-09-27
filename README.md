# AI Agent Studio · Android

原生安卓 App：**Kotlin + Jetpack Compose + Material Design 3**，
**llama.cpp 直接编进 APK**，手机本地跑 `.gguf` 完全离线 —— 不联网、不依赖 Termux、不用开电脑。

云端侧支持 DeepSeek / Kimi / 智谱 / 硅基流动 / OpenAI 一键填写，
以及 Ollama、llama.cpp server、腾讯混元（TC3-HMAC-SHA256 签名）。

## 核心：llama.cpp 是怎么进去的

```
app/src/main/cpp/ggml_jni.cpp   ← JNI 桥（模型加载 / prefill / 采样循环）
        │
        ├─ CMakeLists.txt 把 third_party/llama.cpp 一起 add_subdirectory
        │
        └─ 编出【单个】libggml-jni.so，打进 APK 的 arm64-v8a
                ▲
                │
        Kotlin: LlmEngine.kt 通过 external 方法调用
```

采样链是自己写的，没走 llama.cpp 自带的 sampler：

`logits → 重复惩罚 → temperature → softmax → top-k → top-p → 多项式采样`

还实现了 ChatML 模板拼装和停止词截断。停止词有个坑：token 是分片过来的，
`<|im_end|>` 可能被切成好几段，所以 Kotlin 侧用「缓冲 + 最长前缀保留」
—— 结尾看着像停止词开头的片段先扣住不发，确认不是再吐出去。

多轮对话走增量 prefill：第一轮带上 system + user，
之后每轮只补「上一轮回复 + 新问题」，历史留在 KV cache 里不重算，
比每轮重发整个对话快得多。快撑爆上下文时会自动回退重放最近几轮。

## 编译

### 1. 拉 llama.cpp 源码

```bash
./scripts/fetch-llama-cpp.sh
```

默认拉 master。想固定版本（推荐，避免上游 API 变动）：

```bash
LLAMA_CPP_REF=b6xxx ./scripts/fetch-llama-cpp.sh
```

### 2. 用 Android Studio 打开

需要 **Android Studio Ladybug / Koala 及以上**、**JDK 17**、**NDK 27**。

打开项目根目录，等 Gradle sync 完，直接 Run。

命令行出包：

```bash
./gradlew assembleDebug     # → app/build/outputs/apk/debug/app-debug.apk
./gradlew assembleRelease
```

> 手机上第一次编译 llama.cpp 会比较慢（几分钟到十几分钟），之后有缓存就快了。
> 只出 `arm64-v8a`，32 位跑不动 llama.cpp。

## 怎么用

### 手机本地 GGUF（离线）

1. **模型** 页 → 勾「手机本地 GGUF」→ **选 .gguf**
2. 点 **加载**，等状态变成「已就绪」
3. 回 **对话** 页开聊

推荐模型：

| 模型 | 体积 | 说明 |
| --- | --- | --- |
| **Qwen2.5-1.5B-Instruct-Q4_K_M** | 约 1GB | **默认推荐**，中文好，手机上跑得动 |
| Qwen2.5-0.5B-Instruct-Q4_K_M | 约 0.4GB | 内存吃紧就换这个 |
| Qwen2.5-3B / Llama-3.2-3B Q4_K_M | 约 2GB | 能跑，但明显慢，长对话会吃力 |

模型文件自己下载（HuggingFace，国内访问不了就把 `huggingface.co` 换成 `hf-mirror.com`），
传到手机，在 App 里选一下就导入了（会拷进 App 私有目录）。

**7B 以上不推荐** —— 手机没有独立显存也没有风扇，能加载但慢到几乎没法聊，
连续跑几分钟还会因为发热降频变慢。

### 云端 / 局域网

**模型** 页 → 添加模型源 → 点对应的 chip 一键填写：

| 预设 | 地址 |
| --- | --- |
| DeepSeek | `https://api.deepseek.com/v1` |
| Kimi · 月之暗面 | `https://api.moonshot.cn/v1` |
| 智谱 GLM | `https://open.bigmodel.cn/api/paas/v4` |
| 硅基流动 | `https://api.siliconflow.cn/v1` |
| OpenAI | `https://api.openai.com/v1` |
| Ollama | `http://192.168.1.10:11434`（改成你自己电脑的 IP） |
| llama.cpp server | `http://192.168.1.10:8080/v1` |
| 腾讯混元 | `https://hunyuan.tencentcloudapi.com` |

填完点 **测试** 验一下连通性。

**腾讯混元的密钥比较特殊**：填 `SecretId:SecretKey`，中间一个英文冒号。
签名走 TC3-HMAC-SHA256（见 `HunyuanSigner.kt`）：

```
CanonicalRequest → StringToSign → TC3+SecretKey → date → service → tc3-request → HMAC
```

## 可调参数

**设置** 页（只对本地 GGUF 生效）：

- 上下文长度（512–8192，**改了要重新加载模型**）
- 线程数（默认取核数的一半多点，占满会过热降频）
- Temperature / Top-P / Top-K
- 重复惩罚
- 单条最多生成 token 数

## 界面

Material Design 3：Android 12+ 默认跟随壁纸取色（Material You），
可关掉切固定紫色；深浅色三档（跟随系统 / 浅 / 深）。
底部四个 Tab：对话 / 智能体 / 模型 / 设置。

对话页能渲染 ``` 代码块（没引第三方 Markdown 库，够看就行）。

## 目录

```
app/src/main/
├── cpp/
│   ├── ggml_jni.cpp            JNI 桥 + 自写采样循环
│   └── CMakeLists.txt          链 llama.cpp → libggml-jni.so
├── java/com/excuse2580/aas/
│   ├── MainActivity.kt
│   ├── data/                   ModelSource / Agent / ChatMessage / Prefs
│   ├── engine/
│   │   ├── LlmEngine.kt        本地引擎（ChatML + 停止词 + 增量 prefill）
│   │   ├── CloudEngine.kt      云端 SSE 流式
│   │   └── HunyuanSigner.kt    TC3-HMAC-SHA256
│   ├── vm/AppViewModel.kt
│   └── ui/                     Compose 页面
└── res/
third_party/llama.cpp           ← 跑 fetch 脚本后才有
```

## 已知限制

- 只支持 **arm64 安卓 8.0+**，内存建议 6GB 起（1.5B 模型实测占 1GB 多）
- 自动混合精度/GPU 默认关闭（纯 CPU 最稳）。想试 Vulkan：
  `./gradlew assembleDebug -PggmlVulkan=true`，成不成看手机 GPU 驱动，翻车就关回来
- 手机适合「问一句答一句」的短交互，不适合长文生成
- 云端对话目前是每轮全量重发历史（云端服务无状态），本地模型才是增量的

## 许可

llama.cpp 部分遵循其上游许可；其余代码 MIT。
