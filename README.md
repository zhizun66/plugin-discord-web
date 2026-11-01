# King Box Discord 矩阵营销与私域运营插件（`discord_desktop` & `discord_web`）

本插件是 **King Box** 体系下的独立进程业务插件（基于 `stdio-jsonl` 双向 RPC 通信），采用 **同源开发、双端分轨发布** 架构，完整覆盖 Discord 社媒矩阵的 **安卓模拟器管理、SunnyNet MITM 抓包入库、Android 原生协议栈与 Gateway 网关、服务器成员采集、协议自动加友、ADB+OCR 视觉界面加好友、三栏式实时私信工作台、多租户云端账号中心与云端共享客户池** 全链路业务。

---

## 1. 双端发布轨道与仓库说明

| 轨道标识 (`plugin_id`) | 运行环境 (`target`) | 支持平台 (`platforms`) | 数据库引擎 | 独立 Release 发布仓库 |
| :--- | :--- | :--- | :--- | :--- |
| **`discord_desktop`** | Desktop 客户端 | `windows-amd64` | SQLite (`<DataDir>/discord.db`) | [http://localhost:3000/root/plugin-discord.git](http://localhost:3000/root/plugin-discord.git) |
| **`discord_web`** | Web 服务端控制面 | `linux-amd64`, `windows-amd64` | PostgreSQL (`discord` Schema) | [http://localhost:3000/root/plugin-discord.git](http://localhost:3000/root/plugin-discord.git) |

- **当前版本**：`v0.0.1`（`db_schema_version: 2`）
- **套餐门禁与配额**：内置对接宿主 `license.gate (v1)` 服务，支持 `accounts`（绑定 Discord 账号数，默认上限 500 个）分端占座与租户级原子配额校验。

---

## 2. Desktop 桌面端核心功能手册（`discord_desktop`）

Desktop 端提供 **8 个二级功能模块**（可通过宿主左侧导航菜单或插件顶部导航栏快速切换）：

### 2.1 概览看板（`/`）
- **核心指标实时聚合**：展示本地纳管账号总数、Gateway 在线账号数、模拟器设备数（在线/总数）、本地采集客户池总量与待加目标数、累计协议加友与界面加好友成功数、私信会话数、未读消息数与已成单转化数。
- **今日作业统计**：实时汇总今日新采集成员数、今日协议加友成功/失败数、今日模拟器 UI 加好友成功/失败数及今日私信收发条数。

### 2.2 设备与抓包（`/emulators`）
- **多品牌安卓模拟器统一纳管**：
  - 自动探测并扫描 **雷电模拟器 9（LDPlayer）**、**BlueStacks 5（蓝叠）**、**MuMu 模拟器 12** 实例入库；
  - 支持一键 **启动**、**停止**、**窗口置前唤醒（Win32 TopMost Restore）** 与 **重命名**；
  - 支持为每台模拟器单独绑定宿主代理池（`proxy.v1`）座位，并展示当前模拟器抓包绑定的 Discord 账号。
- **SunnyNet MITM 抓包与无感提号**：
  - 内置 SunnyNet 中间人代理服务（默认端口 `20260`），支持一键启动/停止、导出 PEM 根证书、**通过 ADB 一键将 CA 证书推送安装至模拟器系统证书目录**，以及一键开启/关闭所有在线模拟器的全局代理转发；
  - 实时拦截模拟器内 Discord App 的 HTTPS 与 WebSocket 流量，自动提取 `Authorization` Token、`x-super-properties`、`x-installation-id`、`x-discord-locale`、`x-discord-timezone`、`User-Agent` 等完整 Android 设备指纹，自动入库或更新对应账号。

### 2.3 账号管理（`/accounts`）
- **账号分组与批量导入**：支持创建、重命名、删除账号分组，支持单条创建或按行批量导入 Discord Token（支持 `token` 或 `备注----token` 格式）。
- **Gateway 在线管控与验活**：支持单账号一键上线/下线 Discord Android Gateway 长连接，或调用 `/api/v9/users/@me` 接口验证账号存活状态并同步最新昵称、用户名、头像与个人简介。
- **代理池自动选座与换绑**：深度集成宿主 `proxy.v1` 服务，支持手动指定代理或一键自动分配低负载代理座位。
- **云端同步**：支持单账号或全量一键推送至 Web 控制面（自动维持来源隔离与脱敏展示）。

### 2.4 成员采集（`/collect`）
- **双模式成员列表滚动采集**：
  1. **Gateway 协议直连模式**：使用在线账号的 Gateway 连接发送 `op: 14` 频道成员列表范围订阅帧（`[0,99], [100,199], ...` 步进滚动）；
  2. **真机/模拟器 MITM 隧道复用模式**：当模拟器挂载在 SunnyNet MITM 下时，直接复用模拟器已建立的真实 WebSocket 隧道注入 `op: 14` 订阅帧，零额外连接特征。
- **用户详情补全与云端上传**：监听 `GUILD_MEMBER_LIST_UPDATE` 事件自动过滤机器人并写入本地客户池；后台协程自动拉取用户详情（Bio、Pronouns、Banner、Clan Tag、Nitro 订阅时间），并自动通过 `gatesdk.CloudInvoke` 增量同步至 Web 端云端客户池（`pool.sync`）。

### 2.5 协议加友（`/friend-proto`）
- **精细化过滤与速率风控规则**：
  - 支持开启 **号龄过滤**（基于 Discord Snowflake ID 反算注册时间，过滤新注册小号）；
  - 支持筛选 **Nitro 付费会员**；
  - 支持自定义 **请求间隔（ms）**、**并发执行账号数** 与 **单账号每日加友上限**。
- **云端协同认领**：启动队列后，工作协程优先调用 Web 云端 `pool.claim` 原子认领待加目标（无网络或单机模式下自动回退本地客户池），通过 Android 协议栈发送好友请求，并将结果（成功 / 用户不存在 / 拒收好友请求 / 验证码或频率限制回退锁）实时上报至 `pool.report`。

### 2.6 界面加好友（`/friend-ui`）
- **ADB + UIAutomator + ONNX Runtime（PP-OCR）纯视觉驱动**：
  - 勾选多台安卓模拟器并发执行，自动唤醒模拟器并拉起 Discord App（`com.discord`）；
  - 基于 UIAutomator 控件树与 PP-OCR 文本检测识别，自动点击底部「好友 / 添加好友」入口 $\rightarrow$ 进入「通过用户名添加」页面 $\rightarrow$ 输入可键入的 ASCII 用户名 $\rightarrow$ 点击「发送好友请求」；
  - 自动检测并点击 Discord **Blurple 人类确认弹窗**，实时在表格中展示每台模拟器的当前步骤日志与今日成功/失败计数。

### 2.7 私信聊天（`/chat`）
- **三栏式多账号客服工作台**：
  - **左栏（账号收件箱）**：展示所有账号在线状态与未读私信角标，毫秒级切换当前接待账号；
  - **中栏（会话与好友请求列表）**：支持从 Discord 同步最新好友关系（含 `relation_type=3` 收到的好友申请）与私信会话，支持关键词搜索与状态徽章展示；
  - **右栏（消息时间线与交互）**：
    - 支持一键 **同意好友请求**；
    - 支持发送文本消息与 **≤8MB 图片附件**（自动完成 Discord CDN `attachments` 预签名申请 $\rightarrow$ `PUT` 上传二进制流 $\rightarrow$ 发送带附件引用消息 3 步流程）；
    - 支持 **撤回** 已发送消息；
    - 支持切换客户转化跟进状态（`跟进中 pending` / `已成单转化 converted` / `已关闭 closed`），状态变更自动记录系统审计消息；
    - 支持单条消息一键 **翻译**（对接 `public-libs/llm` 可扩展大模型接口）。

### 2.8 同步中心（`/sync`）
- 集中展示本地账号同步状态（已同步云端数 / 本地总数）、本地采集客户池上传进度（已上传云端数 / 本地总数）与待重试的私信渠道同步队列，支持一键触发全量账号与客户池云端同步。

---

## 3. Web 服务端控制面手册（`discord_web`）

Web 端运行在 `king-box-web` 服务端，使用 PostgreSQL `discord` 独立 Schema 存储数据，严格通过 `runtimedb.VerifyTrustedContextToken` 验签提取可信租户上下文（`TenantID`），提供 **2 个二级管理视图** 与 **11 个 RPC 操作契约**：

1. **账号管理（`/`）**：
   - 多租户云端账号总表（超级管理员全局视角下自动展示「所属租户」列，普通租户严格物理隔离）；
   - 支持接收 Desktop 端 `account.sync` 实时上报、云端新增/删除账号、绑定/自动分配代理座位，并在后台定时自动校验代理座位租约。
2. **云端客户池（`/pool`）**：
   - 集中管理多台 Desktop 终端采集上传的 Discord 目标客户（`discord.discord_pool_users` 表，`tenant_id + user_id` 唯一索引）；
   - 提供 `pool.list`（分页筛选与状态统计）、`pool.sync`（批量幂等入库）、`pool.claim`（多终端并发加友原子认领锁，5 分钟超时自动释放）、`pool.report`（加友结果回写）与 `pool.delete`（清理记录）。

---

## 4. Android 协议栈与安全反风控设计

1. **1:1 还原 Discord Android 协议特征**：
   - 统一使用 `Discord-Android/340013;RNA` 客户端标识与 `okhttp/4.11.0` 网关握手 UA；
   - 严格保持 `[Authorization, Content-Type, User-Agent, X-Super-Properties, X-Discord-Locale, Accept-Language, X-Discord-Timezone, X-Debug-Options, X-Installation-ID, Cookie]` 10 项请求头顺序。
2. **SunnyNet 网络层指纹伪装与代理自愈**：
   - 所有 REST 与 WebSocket 请求优先通过 SunnyNet 注入 Firefox HTTP/2 帧指纹（`H2Fingerprint`）、随机 JA3/TLS 指纹（`SetRandomTLS`）与远程 DNS 解析（`SetDnsServer("remote")`）；
   - 当单账号连续 3 次发生网络超时/连接重置时，自动调用 `proxyclient` 向宿主代理池申请更换新代理并持久化。
3. **Gateway `zstd-stream` 跨帧流式解压**：
   - 连接 `wss://gateway.discord.gg/?encoding=json&v=9&compress=zstd-stream`，内置跨帧流式 `zstd` 字典上下文解码器，完整处理 `op: 10 Hello`、`op: 2 Identify (capabilities: 1734655)`、`op: 1 Heartbeat` 与增量事件分发。
4. **敏感凭据加密与脱敏**：
   - Desktop 本地 SQLite 数据库中的 `TokenEnc` 与 `CookieEnc` 采用基于宿主数据目录派生的 **AES-256-GCM** 密钥加密存储；
   - 前端 UI 展示、跨端云同步（`account.sync`）与全部日志输出仅暴露脱敏后的 `TokenMasked`，严禁明文 Token 落入日志或离开本地受控环境。

---

## 5. 目录结构与常用命令

```text
plugins/discord/
├── apps/
│   ├── desktop/                  # Desktop 桌面端轨道（discord_desktop）
│   │   ├── manifest.json         # 桌面端清单与 38 个 RPC 操作契约声明
│   │   ├── backend/              # Go 后端服务（Gateway/MITM/模拟器/采集/加友/私信/看板）
│   │   └── ui/                   # React + Vite + @king-box/frontend-ui 8 页面工作台
│   └── web/                      # Web 服务端轨道（discord_web）
│       ├── manifest.json         # Web 端清单与账号/客户池 RPC 契约声明
│       ├── backend/              # Go 多租户后端服务（PostgreSQL discord Schema）
│       └── ui/                   # Web 端账号管理与云端客户池前端视图
├── public/
│   └── backend/                  # 双端共享数据模型（GORM Models）、Android 协议包与设备指纹 fp
├── assets/                       # 插件静态图标资源（plugin.svg）
└── Makefile                      # 一键联调、契约校验、双端构建与 GitHub Release 发布脚本
```

### 常用开发与发布命令

```bash
# 1. 启动 Desktop 端独立前后端联调（后端调试网关 :18080 + 前端 Vite :5174）
make dev

# 2. 执行 Go 静态检查与双端 Manifest 契约校验
make vet
make contract
make contract-web

# 3. 构建 Desktop + Web 双端生产包并同步至宿主 data/plugins/ 目录
make build

# 4. 打包双端多平台产物（.zip / .sha256 / .release.json）并发布至各自 GitHub Releases
make package
```
