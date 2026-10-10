# 机器人 OTA 方案（完整定稿）

## 1. 目标与边界

### 1.1 目标

- 业务全部以 Docker 部署、升级、回滚
- **同一套 OTA 程序**覆盖不同本体机型
- **NX 必有**；**x86（运控）按机型可有可无**
- 有网：参与节点按 digest **直拉镜像**；**compose / Job 只由 NX 拉取**，有运控再下发 x86
- 没网：独立 **offline 包**兜底（与 online 包分离）
- 支持公有云、客户私有化内网、纯离线
- 过程可观测：阶段、总进度、已用时；可聚合 Docker pull 进度
- OTA Agent 自身可升级、可回滚；**与业务分版本，按协议兼容约束对齐**

### 1.2 V1 不做（预留）

- 固件、内核 / 宿主机 OS 升级
- 「只回滚失败板、其它板继续」——V1 对**实际参与节点**整 Job 回滚

---

## 2. 硬件与网络

| 板卡 | 是否必有 | 架构 | 网络 | 典型业务 |
|------|----------|------|------|----------|
| **NX** | 必有 | arm64 | 有 WiFi，唯一物理上行 | 感知等 |
| **x86** | 机型决定 | amd64 | 无独立 WiFi；有则经 NX **NAT** 出站（类似虚拟机 NAT / Docker bridge） | 运控等 |

```text
OTA Server + 镜像仓库（云 / 客户内网）
              ▲
              │ WiFi（仅 NX）
┌─────────────┴─────────────┐
│ NX（始终存在）gateway      │  Job/compose 入口、协调、汇总进度与上报
│ ota-agent（systemd）       │  NAT：供 x86（若有）访问仓库
└─────────────┬─────────────┘
              │ 车内网（仅有运控机型）
┌─────────────┴─────────────┐
│ x86（可选）worker          │  收 NX 任务、本机升级、报进度给 NX
│ ota-agent（systemd）       │
└───────────────────────────┘
```

无运控机型：仅 NX，单板模式。

---

## 3. 一套程序，按机型裁剪

```text
实际升级节点 = manifest.nodes ∩ 本机拓扑具备的节点
```

| 来源 | 内容 |
|------|------|
| 出厂拓扑 / 机型 | `model`、`has_x86`，或探测 peer |
| 本次包 / manifest | `nodes`、`required` |

| 情况 | 行为 |
|------|------|
| 仅 NX + 包仅 nx | 只升 NX |
| 有 x86 + 包含 nx、x86 | 两板都升 |
| 有 x86 + 包仅 nx | 只升 NX，运控不动 |
| `x86.required=true` 但无板 | Job 失败 |
| `x86.required=false` 且无板 | 跳过，不报错 |

同一二进制：NX 固定 gateway；有运控才装 x86 worker。

---

## 4. 组件职责

| 组件 | 部署 | 职责 |
|------|------|------|
| OTA Server | 云或客户机房 | 版本、下发 Job、收状态/结果 |
| 镜像仓库 | 云 / 客户 Harbor | 业务镜像；私有化可由 offline 导入 |
| NX agent | NX 宿主机 systemd | 拉包、裁剪节点、本机升级、下发、聚合进度、上报 |
| x86 agent | 仅有运控机型 | 执行本板任务、报进度、本机升级 |

业务进 Docker；**Agent 不进业务 compose**。

状态机等业务逻辑在 **业务容器** 内，随 `app_version` / compose 整包升级与回滚；Agent 不解析状态机内部协议，只负责拉起、切换、健康检查与 OTA 控制面协议。

---

## 5. 两种发布包（必须分开）

在线包**不带**镜像 tar，避免有网下载巨量文件。

### 5.1 在线包

```text
release-app-X.Y.Z-online/
├── manifest.yaml      # 仓库地址 + digest + nodes + agent 兼容声明
├── manifest.sig
├── compose/
│   ├── nx/...
│   └── x86/...        # 仅当本版本需要升运控
└── ota/               # 可选：仅当需要升 agent 时带上
```

### 5.2 离线包

```text
release-app-X.Y.Z-offline/
├── manifest.yaml
├── manifest.sig
├── compose/
│   ├── nx/...
│   └── x86/...
├── images/
│   ├── *-arm64.tar    # NX docker load
│   └── *-amd64.tar    # 仅当升 x86：经车内网传给 x86 load
└── ota/               # 可选：仅当需要升 agent 时带上
```

| 场景 | 包 | 镜像 |
|------|-----|------|
| 仓库可达 | online | 参与节点 `pull@digest` |
| 无网 / 断仓库 | offline | `load`；有 x86 才传 amd64 tar |

Job 标明 `mode: online | offline`。

`*.tar` = `docker save` 归档；arm64 / amd64 不可混用。

**多数发版只有业务**：online 包可不带 `ota/` 目录，Agent 版本保持不变。

---

## 6. Manifest（示意）

```yaml
job_id: ...
app_version: 1.2.3
mode: online
protocol_version: 3              # OTA 控制面协议（manifest / 进度 / NX↔x86 任务格式）
models: [sense-only, sense-control]   # 可选
signature: ...

# Agent 与业务分版本；用兼容声明对齐，不强制同号
agent:
  min_version: "1.4.0"           # 跑本包至少需要的 agent 版本
  # max_version: "2.0.0"         # 可选
  target_version: "1.4.0"        # 可选：本次若要升 agent，目标版本
  # 包内 ota/ 存在且本机 < min_version（或 < target）时，必须先升 agent

nodes:
  - node_id: nx
    arch: arm64
    role: gateway
    required: true
    compose: compose/nx/docker-compose.yml
    compose_hash: sha256:...
    images:
      - name: registry.example.com/perception
        digest: sha256:...

  - node_id: x86
    arch: amd64
    role: worker
    required: false
    compose: compose/x86/docker-compose.yml
    compose_hash: sha256:...
    images:
      - name: registry.example.com/control
        digest: sha256:...

healthcheck:
  timeout_sec: 120
rollback:
  previous_app_version: 1.2.2
  # 若本次同时升了 agent，可记录 previous_agent_version 供按需回滚

# V1 忽略：
# firmware: []
# kernel: null
```

**digest**：镜像内容指纹；验收以 digest 为准，tag 仅展示。

---

## 7. Agent 与业务版本兼容

### 7.1 分版本，不绑死同号

| 对象 | 版本 | 说明 |
|------|------|------|
| 业务（compose / 镜像，含状态机等） | `app_version` | 日常发版主版本 |
| OTA Agent | `agent_version` | **不是每次 OTA 都升** |
| 控制面协议 | `protocol_version` | manifest、进度上报、NX↔x86 任务格式等 |

对齐方式：用 `protocol_version` + `agent.min_version`，**不要**要求 `agent_version == app_version`。

### 7.2 接单时如何判断

1. 本机 agent ≥ `agent.min_version`，且能识别 `protocol_version` → **只升业务**（跳过 agent）
2. 本机 agent < `min_version`，或协议不识别 → 包内必须带 `ota/` agent 包 → **先升 agent，再升业务**
3. 需要升 agent 但包内无 agent 包 / 升失败 → **Job 失败，业务不动**
4. 旧 agent 遇到不认识的 `protocol_version` → **直接拒绝**，禁止半吊子执行

### 7.3 协议演进规矩

1. **能兼容就兼容**：新 agent 尽量仍能执行旧 `protocol_version` / 旧业务包（方便业务单独回退）
2. **不兼容就显式 bump**：改进度字段、任务 JSON、回滚语义等 → `protocol_version + 1`，并抬高 `agent.min_version`
3. **破坏性变更必须同 Job 带 agent**：online/offline 包放入新 agent，顺序为先 agent 后业务

### 7.4 与状态机等业务程序的关系

- 状态机若是业务容器：随 **`app_version` / compose 整包**一起升、一起回
- Agent **不**用自身版本号去对齐状态机内部协议
- Agent 只保证 OTA 控制面协议正确，以及按 compose 拉起/切换/健康检查
- 若将来 agent 需调用状态机专用接口，将该能力纳入 `protocol_version` 或单独 `compat` 字段

### 7.5 回退是否要 agent 与业务一起回

不必每次绑死回退，按 Job 记录的 from/to 决定：

| 情况 | 建议 |
|------|------|
| 本次只升了业务，agent 未变 | 只回滚业务到 `previous_app_version` |
| 本次升了 agent，且新 agent **兼容**旧业务 | 业务回滚即可，agent 可留在新版本 |
| 本次升了 agent，且新 agent **不兼容**旧业务/旧协议 | 回滚时业务与 agent **都回到 Job 记录的 previous** |

Job 落盘建议记录：

```json
{
  "app_from": "2.0.0",
  "app_to": "2.1.0",
  "agent_from": "1.3.0",
  "agent_to": "1.4.0",
  "protocol_version": 3
}
```

---

## 8. 升级流程

### 8.1 在线

1. NX 拉 online 包并验签；**Job 立即落盘**（见 §10.3）
2. 检查 `protocol_version` / `agent.min_version`
3. 若需升 agent：先自升级（落盘 `phase=agent_upgrading`）→ restart → 新进程恢复 Job → 再继续业务
4. 算 `target_nodes`；校验 `required`
5. 含 x86 则下发：compose/x86 + 镜像 digest 列表 + 检查定义
6. 参与节点并行 `pull@digest`（经 Docker API 取进度）
7. 均镜像就绪后按序切换 compose
8. 健康检查 → NX 汇总
9. 全过则锁定并上报成功；任一参与节点失败则按 §7.5 / §13 回滚

### 8.2 离线

1. offline 包放入 NX（inbox / U 盘）
2. 验签 → Job 落盘 → 按需先升 agent → 算 `target_nodes`
3. NX load arm64；若含 x86 则传 amd64 tar + compose
4. 其后切换 / 检查 / 回滚同在线

### 8.3 Compose

- 只由 NX 拉取，保证唯一版本
- 仅 target 含 x86 时下发运控 compose
- x86 不直拉云端 compose

### 8.4 顺序口诀

> **仅当本次需要升 agent 时：先 agent，再业务。**  
> **否则：直接升业务。**

---

## 9. 进度与状态

### 9.1 Job 阶段（phase）

```text
pending → fetching → planning
        → agent_upgrading（可选，按需）
        → distributing（无 x86 跳过）
        → downloading → switching → verifying → succeed

失败：failed / rolling_back → rollback_succeed | rollback_failed
```

未参与节点：`skipped`，不阻塞成功。

### 9.2 对外精简报文（推荐）

```json
{
  "job_id": "…",
  "app_version": "1.2.3",
  "mode": "online",
  "phase": "downloading",
  "progress_percent": 42,
  "elapsed_sec": 185,
  "started_at": "2026-04-10T08:00:00Z",
  "updated_at": "2026-04-10T08:03:05Z",
  "eta_sec": 240,
  "scope": ["nx", "x86"],
  "message": "正在下载镜像",
  "error": null
}
```

| 字段 | 含义 |
|------|------|
| `phase` | 当前阶段 |
| `progress_percent` | 总进度 0–100 |
| `elapsed_sec` | 已进行时长（由 `started_at` 计算，agent 重启后连续） |
| `started_at` / `updated_at` | 开始与最近更新 |
| `eta_sec` | 可选，预估剩余 |
| `scope` | 实际参与节点 |
| `message` | 简短说明 |
| `error` | 失败时：`node_id` / `phase` / `code` / `message` |

分节点明细默认留在 NX 内部算总进度；需要时再开详情查询。

### 9.3 总进度权重（可配）

| 阶段 | 权重 |
|------|------|
| fetching | 0–5% |
| agent_upgrading | 5–10%（跳过则并入后续） |
| distributing | 10–15%（可跳过） |
| downloading | 15–70% |
| switching | 70–85% |
| verifying | 85–100% |

多节点下载进度：平均或按镜像体积加权后写入总百分比。

### 9.4 Docker pull 进度

- 使用 **Docker Engine API**（`/images/create` 进度流）或官方 SDK 回调
- 用各层 `progressDetail.current/total` 汇总单镜像 → 节点 → Job
- 层已存在会跳过，进度可能跳变（正常）
- 离线 `load`：按文件传输字节 + load 完成计 `downloading`

### 9.5 上报路径

```text
x86（若有）──进度/状态──► NX 聚合 ──► Server / 车载或运维 UI
NX 本机进度 ─────────────►
```

- 状态变化立即推；下载进度节流（如 1s 或每 5%）
- 离线 Job 写 `/var/ota/status/current.json`，有网再补传或仅现场查

---

## 10. Agent 部署、自升级与 Job 持久化

### 10.1 安装布局

```text
/usr/local/ota/
├── current -> releases/a.b.c/ota-agent
├── releases/
│   ├── a.b.b/ota-agent
│   └── a.b.c/ota-agent
└── state.json              # agent 自身 trying / confirmed

/var/ota/jobs/<job_id>/
├── manifest.yaml           # 完整任务（重启后仍在）
├── state.json              # phase / next / from-to 版本
└── ...
```

```yaml
# NX
role: gateway
node_id: nx
model: sense-control
topology: { has_x86: true }
server: https://ota.example.com
bundle_dir: /var/ota/inbox
peer:
  - node_id: x86
    addr: https://192.168.x.x:9100

# x86（仅有运控时）
role: worker
node_id: x86
listen: 0.0.0.0:9100
trust_gateway_only: true
```

### 10.2 Agent 自升级（按需）

- **不是每次业务 OTA 都升 agent**
- 仅当包声明需要且本机不满足 `min_version`（或指定了 `target_version`）时执行
- 流程：验签 → 写入新 `releases/` → 切 `current` → `systemctl restart` → 自检 → `confirmed`
- 失败：回 `prev`；systemd/watchdog 兜底

### 10.3 重启后如何继续升业务（Job 落盘）

Agent 自升级会重启进程，**内存中的“下一步”会丢**，因此：

1. 接单后 **先落盘** Job（manifest + `state.json`）
2. 进入 agent 升级前写入例如：`phase=agent_upgrading`，`next=download_business`
3. restart 后新进程启动扫描 `/var/ota/jobs/*/state.json`
4. 若发现未完成 Job：本机 agent 已达目标且自检通过 → 从 `next` 继续业务升级
5. 若 agent 未起来或版本不对 → 回滚 agent，Job 失败（业务未切换）
6. 全程阶段更新都写盘；成功或最终失败后归档/清理

启动恢复要点：幂等（按 phase 判断，避免重复切栈）；`elapsed_sec` 用落盘的 `started_at` 计算。

`state.json` 示意：

```json
{
  "job_id": "…",
  "phase": "agent_upgrading",
  "next": "download_business",
  "app_from": "2.0.0",
  "app_to": "2.1.0",
  "agent_from": "1.3.0",
  "agent_to": "1.4.0",
  "protocol_version": 3,
  "started_at": "…"
}
```

---

## 11. 公有云 / 私有化 / 离线

| 形态 | Server | 镜像 | 机器人 |
|------|--------|------|--------|
| 公有云 | 托管 | 云仓库 online 直拉 | NX 配云端 URL |
| 私有化 | 客户内网 Server | offline 导入 Harbor 后 online | 改 server/证书/registry |
| 纯隔离 | 可无 | 仅 offline | NX 扫 inbox |

协议同一套；x86 有无由机型决定，与部署形态无关。

---

## 12. 安全

- 包 / manifest / agent 签名校验
- worker 只信 NX 签名任务
- 镜像按 digest 验收
- 私有化可换 CA / 设备证书
- online / offline 包分离
- 不识别的 `protocol_version` 拒绝执行

---

## 13. 失败与回滚（V1）

| 对象 | 策略 |
|------|------|
| 业务 | 参与节点任一失败 → 参与节点回滚到 `previous_app_version`；回滚过程同样报 `phase` / 进度 / 时长 |
| 未参与节点 | 不动 |
| Agent（本次未升） | 不动 |
| Agent（本次已升且兼容旧业务） | 业务回滚时 agent 可保留新版本 |
| Agent（本次已升且不兼容旧业务） | 与业务一并回 `agent_from` |
| Agent 自升级失败 | 回 `prev`，业务未切换 |
| 拉取失败 | 不切换，维持当前版本 |

---

## 14. 扩展预留

`firmware` / `kernel` 字段 V1 忽略；日后按节点加 handler，不改 NX 中心 + 拓扑裁剪框架。

---

## 15. 实现优先级

1. NX 单板 online：拉包 → pull（API 进度）→ 切 compose → 检查 → 回滚 + 精简状态上报 + Job 落盘
2. 拓扑裁剪（`has_x86` + `required`）
3. 有 x86：compose 下发 + 双板直拉 + 进度聚合 + 整 Job 回滚
4. `protocol_version` / `agent.min_version` 兼容检查
5. 按需 agent 自升级 + 重启后 Job 恢复
6. offline 模式
7. Server 对接（云 / 私有化）
8. firmware/kernel 空 handler

---

## 16. 一句话总结

> **同一套 OTA，以 NX 为唯一控制面；按机型与包决定只升 NX 还是连同 x86；有网参与节点 digest 直拉、compose 由 NX 下发；没网用独立 offline 包；Agent 与业务分版本，用 protocol/min_version 约束兼容，按需才升 agent（先落盘再重启，再续业务）；对外上报阶段、总进度、已用时；固件/内核以后再接。**

---

## 17. 建议开工接口清单

1. `manifest` 规范（online/offline、`required`、机型、`protocol_version`、`agent.min_version`）
2. 本机 `topology` / `model` 配置
3. `NX → x86` 任务 JSON（无 x86 则无）
4. 对外 `JobStatus` 精简字段（本文 §9.2）
5. Agent 状态机（单板/双板、进度聚合、`agent_upgrading`、Job 恢复）
6. Job 落盘目录与 `state.json` 字段
7. online/offline 与「仅 NX / NX+x86」构建矩阵
8. systemd / 目录 / 按需自升级规范

本文可作为评审与实现基线。
