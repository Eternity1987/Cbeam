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
- OTA Agent 自身可升级、可回滚

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

---

## 5. 两种发布包（必须分开）

在线包**不带**镜像 tar，避免有网下载巨量文件。

### 5.1 在线包

```text
release-app-X.Y.Z-online/
├── manifest.yaml      # 仓库地址 + digest + nodes
├── manifest.sig
├── compose/
│   ├── nx/...
│   └── x86/...        # 仅当本版本需要升运控
└── ota/               # 可选：agent 包
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
└── ota/
```

| 场景 | 包 | 镜像 |
|------|-----|------|
| 仓库可达 | online | 参与节点 `pull@digest` |
| 无网 / 断仓库 | offline | `load`；有 x86 才传 amd64 tar |

Job 标明 `mode: online | offline`。

`*.tar` = `docker save` 归档；arm64 / amd64 不可混用。

---

## 6. Manifest（示意）

```yaml
job_id: ...
app_version: 1.2.3
mode: online
models: [sense-only, sense-control]   # 可选
signature: ...

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

# V1 忽略：
# firmware: []
# kernel: null
```

**digest**：镜像内容指纹；验收以 digest 为准，tag 仅展示。

---

## 7. 升级流程

### 7.1 在线

1. NX 拉 online 包并验签
2. 算 `target_nodes`；校验 `required`
3. 含 x86 则下发：compose/x86 + 镜像 digest 列表 + 检查定义
4. 参与节点并行 `pull@digest`（经 Docker API 取进度）
5. 均镜像就绪后按序切换 compose
6. 健康检查 → NX 汇总
7. 全过则锁定并上报成功；任一参与节点失败则参与节点整 Job 回滚

### 7.2 离线

1. offline 包放入 NX（inbox / U 盘）
2. 验签 → 算 `target_nodes`
3. NX load arm64；若含 x86 则传 amd64 tar + compose
4. 其后切换 / 检查 / 回滚同在线

### 7.3 Compose

- 只由 NX 拉取，保证唯一版本
- 仅 target 含 x86 时下发运控 compose
- x86 不直拉云端 compose

---

## 8. 进度与状态

### 8.1 Job 阶段（phase）

```text
pending → fetching → planning → distributing（无 x86 跳过）
        → downloading → switching → verifying → succeed

失败：failed / rolling_back → rollback_succeed | rollback_failed
```

未参与节点：`skipped`，不阻塞成功。

### 8.2 对外精简报文（推荐）

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
| `elapsed_sec` | 已进行时长 |
| `started_at` / `updated_at` | 开始与最近更新 |
| `eta_sec` | 可选，预估剩余 |
| `scope` | 实际参与节点 |
| `message` | 简短说明 |
| `error` | 失败时：`node_id` / `phase` / `code` / `message` |

分节点明细默认留在 NX 内部算总进度；需要时再开详情查询。

### 8.3 总进度权重（可配）

| 阶段 | 权重 |
|------|------|
| fetching | 0–5% |
| distributing | 5–10%（可跳过） |
| downloading | 10–70% |
| switching | 70–85% |
| verifying | 85–100% |

多节点下载进度：平均或按镜像体积加权后写入总百分比。

### 8.4 Docker pull 进度

- 使用 **Docker Engine API**（`/images/create` 进度流）或官方 SDK 回调
- 用各层 `progressDetail.current/total` 汇总单镜像 → 节点 → Job
- 层已存在会跳过，进度可能跳变（正常）
- 离线 `load`：按文件传输字节 + load 完成计 `downloading`

### 8.5 上报路径

```text
x86（若有）──进度/状态──► NX 聚合 ──► Server / 车载或运维 UI
NX 本机进度 ─────────────►
```

- 状态变化立即推；下载进度节流（如 1s 或每 5%）
- 离线 Job 写 `/var/ota/status/current.json`，有网再补传或仅现场查

---

## 9. 公有云 / 私有化 / 离线

| 形态 | Server | 镜像 | 机器人 |
|------|--------|------|--------|
| 公有云 | 托管 | 云仓库 online 直拉 | NX 配云端 URL |
| 私有化 | 客户内网 Server | offline 导入 Harbor 后 online | 改 server/证书/registry |
| 纯隔离 | 可无 | 仅 offline | NX 扫 inbox |

协议同一套；x86 有无由机型决定，与部署形态无关。

---

## 10. Agent 部署与自回滚

```text
/usr/local/ota/
├── current -> releases/a.b.c/ota-agent
├── releases/
│   ├── a.b.b/ota-agent
│   └── a.b.c/ota-agent
└── state.json              # trying / confirmed
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

自身升级：验签 → 新版本目录 → 切 `current` → restart → 自检 → `confirmed`；失败回旧版。

systemd/watchdog 兜底。建议 **先升 agent，再升业务**。

---

## 11. 安全

- 包 / manifest / agent 签名校验
- worker 只信 NX 签名任务
- 镜像按 digest 验收
- 私有化可换 CA / 设备证书
- online / offline 包分离

---

## 12. 失败与回滚（V1）

| 对象 | 策略 |
|------|------|
| 业务 | 参与节点任一失败 → 参与节点全部回滚到 `previous_app_version`；回滚过程同样报 `phase` / 进度 / 时长 |
| 未参与节点 | 不动 |
| Agent | 留旧二进制，失败回 `prev` |
| 拉取失败 | 不切换，维持当前版本 |

---

## 13. 扩展预留

`firmware` / `kernel` 字段 V1 忽略；日后按节点加 handler，不改 NX 中心 + 拓扑裁剪框架。

---

## 14. 实现优先级

1. NX 单板 online：拉包 → pull（API 进度）→ 切 compose → 检查 → 回滚 + 精简状态上报
2. 拓扑裁剪（`has_x86` + `required`）
3. 有 x86：compose 下发 + 双板直拉 + 进度聚合 + 整 Job 回滚
4. offline 模式
5. Server 对接（云 / 私有化）
6. Agent 自升级
7. firmware/kernel 空 handler

---

## 15. 一句话总结

> **同一套 OTA，以 NX 为唯一控制面；按机型与包决定只升 NX 还是连同 x86；有网参与节点 digest 直拉、compose 由 NX 下发；没网用独立 offline 包；对外上报阶段、总进度、已用时（pull 进度由 Docker API 在板端汇总）；Agent 宿主机常驻可自回滚；固件/内核以后再接。**

---

## 16. 建议开工接口清单

1. `manifest` 规范（online/offline、`required`、机型）
2. 本机 `topology` / `model` 配置
3. `NX → x86` 任务 JSON（无 x86 则无）
4. 对外 `JobStatus` 精简字段（本文 §8.2）
5. Agent 状态机（单板/双板、进度聚合）
6. online/offline 与「仅 NX / NX+x86」构建矩阵
7. systemd / 目录 / 自升级规范

本文可作为评审与实现基线。
