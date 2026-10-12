# LUNA

一个 Android 浏览器。搜索集成、就当前页提问、网络层查证，全部在本机跑，不连任何服务器。

```
┌─────────────────────────────────────────────┐
│  Android 壳（WebView + 原生控件）             │
│      │                                       │
│      │ http://127.0.0.1:8800                 │
│      ▼                                       │
│  Node 引擎（打包在 APK 里，开机自启）          │
│      ├─ 搜索    解析引擎结果，用我们自己的界面渲染 │
│      ├─ 查证    DNS / TLS / 证书 / 第三方资源    │
│      └─ 提问    抓正文 → 嵌入 → 检索 → 本地模型   │
│                        │                     │
│                        ▼                     │
│              llama.cpp（OpenCL / CPU）        │
└─────────────────────────────────────────────┘
```

## 它和别的浏览器有什么不一样

搜索不用别人的页面。输入词之后打开的是 LUNA 自己渲染的结果页，搜索引擎只是数据源。
换引擎时界面不变，用户也不用看推广和侧栏。

「这一页可信吗」是一个能回答的问题。普通浏览器只能告诉你连接是安全的。
LUNA 会做一遍网络层测量：DNS 解析一不一致、证书覆不覆盖这个域名、
有没有被降级到明文、页面加载了多少第三方追踪。问它这页可信吗，答案里会带上这些。

提问只用本机的模型。没有 API key，没有账号，页面内容不出设备。

依赖分得清。项目 88% 是自己写的，外部依赖只有三个库（Node、llama.cpp、OpenSSL），
它们在本机跑，不会哪天被墙。唯一依赖外部服务的是搜索引擎，而它是可替换的：
选了连不上的引擎时，结果页会把现在能用的几个摆出来让你点。
即使所有搜索引擎都连不上，查证和提问照样能用。

别的工具能直接用它的模型。引擎暴露一套 OpenAI 兼容的接口，
手机上任何一个支持自定义 base_url 的编辑器插件、笔记应用、命令行工具，
都能把 LUNA 的引擎当后端。模型和算力都在本机，不出设备。

```bash
curl http://127.0.0.1:8800/v1/models
curl http://127.0.0.1:8800/v1/chat/completions -H 'content-type: application/json' \
  -d '{"messages":[{"role":"user","content":"你好"}],"stream":true}'
curl http://127.0.0.1:8800/v1/embeddings -H 'content-type: application/json' \
  -d '{"input":["一段文字"]}'
```

只绑 127.0.0.1，但同设备上的其他应用能访问 localhost。默认不设 key 是为了开箱能用；
要挡的话设 `LUNA_API_KEY`，之后所有 `/v1/*` 都要 `Authorization: Bearer <key>`。

## 它自己带着维护文档

引擎内置了一份 `/skill`，内容是「在 Android 上跑 llama.cpp 会遇到什么」：

```bash
curl http://127.0.0.1:8800/skill
```

OpenCL 比 CPU 快多少、Vulkan 为什么不能用、`-fa 0` 为什么是硬要求、
内存为什么必须看 `MemFree` 而不是 `MemAvailable`、退化弧是怎么回事。
这些在别处查不到。大厂写在内部文档里，闭源项目不写。

源文件是 [`.dsh/skills/luna-maintain/SKILL.md`](.dsh/skills/luna-maintain/SKILL.md)
（同时也是 DSH 技能，AI 助手会直接读到），构建时剥掉 frontmatter 打进 APK。

## 功能清单

### 浏览器

这些是地基，少一个用户就会觉得这是坏的：

    多标签 · 前进/后退/刷新/停止 · 地址栏 · 集成搜索
    下载 + 下载管理 · 上传文件 · 摄像头/麦克风 · 定位
    证书错误处理 · 弹窗拦截 · 会话恢复（关掉再开标签还在）
    历史记录 · 书签 · 页内查找 · 分享
    长按菜单（新标签打开 / 复制链接 / 保存图片）
    标签管理 · 恢复关闭的标签
    无痕 · 桌面版网站 · 站点权限记忆
    夜间模式跟随系统 · 强制深色网页
    拦弹窗和跳转（window.open · 跳应用商店 · 全屏遮罩）

### 这台设备上独有的

    就这一页提问（RAG，本地模型，答案带来源）
    查证（证书 / DNS / 追踪器，纯网络层，不碰模型）
    整页翻译（本地小模型）
    阅读模式（去掉广告和侧栏，只留正文）
    OpenAI 兼容接口（别的工具能把它当后端用）
    模型后端可换（本机 / 局域网 / 任意云端）

## 明确不做的

写在这里，是为了以后不要再顺手补上。

    手势（下拉刷新 / 边缘后退）
       实测经验：上网这么多年它只帮了倒忙。
       下拉刷新在你滚到顶部想再看一眼时触发；
       边缘后退和你正在操作的抽屉、轮播抢。
       两者都是帮忙帮出来的麻烦，而且阈值只能靠反复试。

    首页快捷方式
       不好看，也用不上。手机浏览器的主页是输入框，不是门户。

    同步（书签 / 历史跨设备）
       要账号体系，而这个项目的立身之本是数据不出设备，两者矛盾。
       真要做的话得先想清楚是什么形态。

    扩展系统
       看过 Nyxt 的说法：浏览器插件受沙箱限制，很多事根本做不到。
       LUNA 的引擎在页面沙箱外，本来就不受那条线约束。
       但能扩展和有扩展生态是两件事，后者要几年。

## 已知的问题（没解决）

    100 秒一次问答   本机模型跑不动。换云端就失去了数据不出设备的意义。
                     这是当前最大的产品风险，属于产品定位问题而不是工程问题。
    签名是测试密钥    pass:android。真发布要换。
    API key 明文存    provider.json。
    /v1/* 默认无鉴权  同设备上任何应用都能读你的对话。设 LUNA_API_KEY 可以挡。

## 目录

```
luna/            引擎（Node）
  server.mjs       HTTP 服务（8800）
  search.mjs       搜索结果解析
  dossier.mjs      网络层查证
  rag.mjs          网页问答（检索 + 生成，流式）
  openai.mjs       OpenAI 兼容接口（/v1/models /v1/chat/completions /v1/embeddings）
  engines-health.mjs 搜索引擎可达性（连不上时给出可用的）
  llm.mjs          llama.cpp 调度（嵌入 / 对话，OpenCL 优先）
  models.mjs       模型发现
  public/          前端（首页 / 结果页 / 面板）
engine/          网络测量内核（从 NetSpace 冻结内核取的）
apk/             Android 壳
  src/net/luna/    Java
  res/             资源（图标是生成的，见 tools/）
  build.sh         构建脚本
tools/           构建、渲染、验证工具
docs/            设计文档和实测数据
```

## 构建

需要一份 Android 工具链（`aapt2` / `javac` / `d8` / `apksigner`）。
不用 Gradle，不用 androidx，不用 Material，界面全是 framework 原生控件。

```bash
bash apk/build.sh
# → LUNA.apk + LUNA-<版本>-安装副本.zip
```

脚本按这个顺序找工具链：环境变量 `PREFIX_USR`，然后 `apk/tools/prefix/`，
再然后隔壁 NetSpace 的 `tools/prefix/`。`ANDROID_JAR` 同理。

### 几个构建上的坑（都踩过）

依赖闭包要自检。引擎模块是一堆 `.mjs`，漏拷一个，构建照样成功，
装到手机上才 `ERR_MODULE_NOT_FOUND`。构建脚本会从入口开始走一遍
import 图，缺了就报错并停。

版本号每次构建必须自增。应用把 assets 解压到私有目录，靠版本号
判断要不要重新解压。版本号不变的话，覆盖安装后网页内容还是旧的，
而启动器图标是 APK 资源、会更新，于是现象是图标变了但界面行为没变，
很难往这上面想。`tools/bumpver.mjs` 每次构建自增，脚本末尾还会断言
清单版本等于包内版本。

`LUNA.apk` 会被安装器删掉。装完就没了，所以同时产一份 `.zip` 副本，
安装器不认这个后缀，不会碰。

## 验证

这个项目的开发环境看不到运行时界面。没有浏览器，没有设备屏幕，
用户只能用截图反馈。所以大部分验证是离线的、自动化的：

```bash
node tools/tdzcheck.mjs      # 模块顶层先用后声明（会白屏）
node tools/pagecheck.mjs     # 页面引用的文件、模块类型、id
node tools/svgcheck.mjs      # 渲染生成的 SVG，和几何定义逐像素比
node tools/metrics.mjs       # 图标尺寸和居中度
node tools/panelcheck.mjs    # 面板逻辑（DOM 桩，真跑模块）

# 端到端
node tools/serve.mjs start   # 起一个脱离会话的引擎（不会被工具调用带走）
node tools/accept.mjs        # 每个界面走的接口 + 前端真正消费的形状
node tools/accept.mjs --with-model   # 加上提问链路（要起模型，几分钟）
node tools/apkcheck.mjs      # 包里的就是当前源码吗
node tools/smoke.mjs         # 从 APK 冷启动走一遍（模拟首次安装）

# 硬件
node tools/mem.mjs --watch   # 内存（跑重活之前先看）
node tools/memsteps.mjs      # 分步量模型内存
node tools/gpubench.mjs      # OpenCL vs CPU
```

每个检查器都拿已知的真 bug 验证过能抓到。一个在它专门要抓的 bug 上给假阴性的
检查器，比没有检查器更危险。`tdzcheck` 第一版就是这样：我造了个真 TDZ 去测它，
它报通过。原因是声明行被整行跳过，而 `export const A = B_CONST;`
是一边声明一边使用。

检查器不能改写被测代码。`panelcheck` 早期为了在 Node 里跑，
会把 `import` 行剥掉再执行，结果正好挡住了「页面用普通 `<script>` 加载模块」
这类错误。现在它用真的动态 `import()`。

> 完整的维护纪律（踩过的坑、平台事实、LLM 实测）在
> [`.dsh/skills/luna-maintain/SKILL.md`](.dsh/skills/luna-maintain/SKILL.md)，
> 它同时是一份 DSH 技能，AI 助手会直接读到。

## 界面

图标只有白和黑，颜色由 CSS / `ColorFilter` 决定，状态用角标圆点表示。
一份几何定义（`tools/icondata.mjs`）出三种产物：网页 SVG、
Android vector drawable、启动器 PNG。

不用 emoji。emoji 字形来自系统字体，同一个字符在荣耀、小米、三星上
长得完全不一样，粗细和基线也对不齐，做不出统一的界面。

设计稿可以离线出，不需要设备：

```bash
node tools/uimock3.mjs main      # 界面
node tools/uimock3.mjs dossier   # 网络档案
node tools/iconsheet.mjs         # 图标（实尺和放大）
```

## 运行

需要设备上有一份 gguf 模型。默认：

- 嵌入：`embeddinggemma-300m-Q4_0.gguf`（265 MB，768 维）
- 对话：非推理类、1.2~2.2 GB 那一档（`MiniCPM5-2B` 或 `Qwen3-1.7B`）

推理模型（R1 蒸馏 / Thinking 类）不适合做检索问答。它们的输出是思考过程，
一个「用一句话说明」能想 30 秒，输出预算被思考吃光，真正的答案经常被截断。
`pickDefaults()` 会排除它们。

模型不打包进 APK（本机现成的一整套在 `/sdcard/models/`，再塞一份几 GB 没意义）。
用系统文件选择器（SAF）选一次，复制到应用目录，不需要任何存储权限。

## 实测数据

模型速度、OpenCL vs CPU、内存占用、RAG 耗时，全部在
[`docs/MEASUREMENTS.md`](docs/MEASUREMENTS.md)，每条都注明了测试条件。

几条结论先摆在这里：

- 这台设备的 Vulkan 后端是坏的（Adreno 驱动问题），OpenCL 可用且快 38%
- 手机上是内存带宽瓶颈而不是算力瓶颈：`-t 2` 比 `-t 4` 快，4.9 GB 的 MoE 只比 1.7B 慢一点
- 不用的模型必须卸掉：两个模型合占 2.2 GB，而设备空闲内存常年只有几百 MB
- OpenCL 的单次分配上限是 1 GB，模型大了会分配失败，所以必须能退回 CPU

## 许可

双许可，见 [LICENSE](LICENSE)。

```
个人使用、学习、研究、修改、分享   免费（PolyForm Noncommercial 1.0.0）
商业使用                          需单独授权（luna.license@qq.com）
```

不是 MIT。原因：MIT 允许别人把 LUNA 一行不改打包上架、加广告，
而原作者拿不到任何东西。AGPL 也解决不了，它允许商业使用，只要求改了开源，
而想摘桃子的人不怕开源。

内核 SOID 仍是 MIT。那个分工是故意的：内核要宽松，产品要保留。
