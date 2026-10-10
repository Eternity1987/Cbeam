# 机器人 OTA 方案（架构基线）

> **文档定位**：本文是可评审、可开干的**架构基线**，不是量产定稿。  
> 总体技术方向正确可保留；须补齐异常世界（断电、断联、门禁、清理、事务、健康契约等）后，方可作为正式实现基线。  
> 文中协议字段、JSON 示例为**规范意图/伪代码**，实现时可微调字段名，但语义应保持一致。

---

## 1. 目标与边界

### 1.1 目标

- 业务全部以 Docker 部署、升级、回滚
- **同一套 OTA 程序**覆盖不同本体机型
- **NX 必有**；**x86（运控）按机型可有可无**
- 有网：参与节点按 digest **直拉镜像**；**Release/Job 由 NX 拉取与协调**，有运控再下发 x86
- 没网：独立 **offline 包**兜底（与 online 包分离）
- 支持公有云、客户私有化内网、纯离线
- 过程可观测：**下载进度**与**升级进度**分开；含阶段、百分比、已用时
- OTA Agent 与业务 / 控制面协议**分版本**；按兼容约束对齐
- 异常可收敛：断电恢复、双板失联、健康失败、空间不足、门禁拒绝等

### 1.2 明确不做 / 延后

| 范围 | 说明 |
|------|------|
| MCU 固件 | 分类已定义；V1 只留 Executor 接口，不实现刷写 |
| NX/x86 内核、完整 OS、Bootloader | 同上，单独通道，V1 不实现 |
| Agent 自升级 | **延后到 V1.3**；V1.0–1.2 靠装机预装满足 `min_agent` 的 Agent，不满足则拒绝 Job |
| 「只回滚失败板」 | 参与节点必须整 Job 收敛到同一 release |

### 1.3 必须保留的骨架

- NX 中心化协调；可选 x86 worker；同一 Agent 按 role 运行
- Agent 独立于业务 compose，systemd 常驻
- online / offline 分层；镜像按 digest
- Agent / 业务 / 协议解耦
- Job 落盘恢复
- 按层升级：Release 没声明的层不升级、不重启

### 1.4 升级分层：一次 Job 只升声明过的层

**不是每次 OTA 都升业务 + Agent + MCU + 内核。** Release 只包含本次要变的组件；没有的层不动、不重启。

| 层 | 典型内容 | 生效方式 | 频率 | 本基线 |
|----|----------|----------|------|--------|
| **业务** | Docker 镜像、Compose、业务配置 | 重建/切换容器；**不重启** NX/x86 | 最高，日常发版 | **V1.0–1.2 只做这一层** |
| **OTA Agent** | Agent 二进制 | 重启 Agent 进程；**不重启宿主机** | 协议变化才升 | V1.3 |
| **MCU 固件** | 关节、电源、传感器等 MCU | 重启**对应 MCU**；通常不重启 NX/x86 | 中低 | 预留 Executor |
| **板卡系统** | NX/x86 内核、完整 OS、Bootloader | **重启对应板卡**（常为 A/B 分区） | 最低 | 预留 Executor |

规则：

1. Job 的重启范围 = 本次 Release 里**实际出现**的层的并集，取最重的那一档。只有业务时，整机不重启。
2. 各层版本独立。业务 `release_id`、Agent 版本、MCU 固件版本、OS 版本互不绑死。
3. 回滚也按层：业务回 `active_before`；未升级的层不动。
4. 板卡系统升级与业务升级不要打进同一次默认 Job。系统层失败窗口包含整板重启，和 Docker 切换不是同一套事务。
5. 安全门禁仍在「会停业务或会动执行器」的激活前执行。只下载、只升 Agent 进程时，不要求整机进入业务安全态；MCU/系统层激活前必须过门禁。

```text
日常：   只含 business        → 切容器，板子不重启
偶发：   business + agent     → 先重启 Agent 进程，再切容器
更少：   mcu                  → 刷指定 MCU 并复位该 MCU
极少：   os / kernel / boot   → 单独 Job，重启该板卡
```

---

## 2. 硬件与网络

| 板卡 | 是否必有 | 架构 | 网络 | 典型业务 |
|------|----------|------|------|----------|
| **NX** | 必有 | arm64 | 有 WiFi，唯一物理上行 | 感知等 |
| **x86** | 机型决定 | amd64 | 无独立 WiFi；有则经 NX **NAT** 出站 | 运控等 |

```text
OTA Server + 镜像仓库（云 / 客户内网）
              ▲
              │ WiFi（仅 NX）
┌─────────────┴─────────────┐
│ NX（始终存在）gateway      │  Job/Release 入口、事务协调、进度汇总、上报
│ ota-agent（systemd）       │  NAT：供 x86（若有）访问仓库
└─────────────┬─────────────┘
              │ 车内网（仅有运控机型）
┌─────────────┴─────────────┐
│ x86（可选）worker          │  验厂商签名、执行本板事务步骤、报进度
│ ota-agent（systemd）       │
└───────────────────────────┘
```

业务层不做「只升其中一块板」。板卡是否参与，由机型决定，不由单个 Release 挑着升。

| 机型 | 业务 Release 必须包含 | 行为 |
|------|------------------------|------|
| 无运控（只有 NX） | 仅 nx | 单板事务 |
| 有运控（NX + x86） | **nx 和 x86 一起** | 双板同一 `release_id`，一起 PREPARE / ACTIVATE / COMMIT / ROLLBACK |
| 有运控但 Release 只有一块板 | — | **拒绝 Job**，不进入切换 |

四层（业务容器、OTA Agent、MCU/固件、板卡 OS）仍然分开。一次 Job 只含声明的层；但**业务层只要这台车有两块板，两块板必须同一版本**。MCU、OS、Agent 仍按各自通道升级，不跟每次业务包绑在一起。

---

## 3. 组件职责

| 组件 | 部署 | 职责 |
|------|------|------|
| OTA Server | 云或客户机房 | 登记 Release、创建 Job、收状态 |
| 镜像仓库 | 云 / Harbor | 业务镜像；私有化可由 offline 导入 |
| NX agent | 宿主机 systemd | 协调、门禁、本机升级、下发、汇总 |
| x86 agent | 仅有运控机型 | 验签、本机升级、trial lease、回报 |
| 业务状态机 | 业务容器 | 通过 **PrepareForOTA** 提供门禁，不负责执行 OTA |
| ota-bootstrap | 宿主机（V1.3） | Agent 起不来时切回 previous（V1.0 不实现自升级） |

业务进 Docker；**Agent 不进业务 compose**。  
Agent **不解析**状态机内部业务协议，但必须通过标准门禁接口确认「允许停机升级」。

---

## 4. Release Manifest 与 Job Document（必须拆开）

### 4.1 Release Manifest（不可变、厂商签名、可复用）

同一 Release 可发给多台同机型机器人，不为每个 Job 重签内容包。无运控机型的 Release 只有 `nx`。有运控机型的业务 Release 必须同时带 `nx` 和 `x86`，两板 `required: true`。

```json
{
  "release_id": "q01-1.2.3",
  "product": "dr-robot",
  "models": ["sense-only", "sense-control"],
  "protocol_version": 3,
  "agent": { "min_version": "1.4.0" },
  "created_at": "...",
  "expires_at": "...",
  "nodes": [
    {
      "node_id": "nx",
      "arch": "arm64",
      "platform": "linux/arm64",
      "required": true,
      "start_order": ["system", "driver"],
      "stop_order": ["driver", "system"],
      "compose_project_name": "dr-robot-nx",
      "artifacts": [
        {
          "name": "compose-bundle",
          "url": "https://.../nx-compose.tgz",
          "sha256": "...",
          "compressed_size": 123,
          "unpacked_size": 456
        }
      ],
      "images": [
        {
          "repository": "registry.example.com/perception",
          "digest": "sha256:...",
          "platform": "linux/arm64",
          "estimated_size": 800000000
        }
      ],
      "healthchecks": {},
      "trial_deadline_sec": 600
    },
    {
      "node_id": "x86",
      "arch": "amd64",
      "required": true,
      "compose_project_name": "dr-robot-x86",
      "start_order": ["system", "driver"],
      "stop_order": ["driver", "system"],
      "images": [],
      "healthchecks": {},
      "trial_deadline_sec": 600
    }
  ],
  "healthchecks": {},
  "space": {
    "estimated_download_bytes": 0,
    "estimated_unpacked_bytes": 0
  }
}
```

签名：对 **canonical JSON（或原始字节）** 签名；`manifest.sig` **外置**，Manifest 内不放可重排的 `signature` 字段。  
签名覆盖：`release_id`、时间窗、机型、compose/config 整包 hash、全部镜像 digest、agent 要求、health 策略、offline tar hash（若有）等。

### 4.2 Job Document（某次发布任务）

```json
{
  "job_id": "job-10027",
  "release_id": "q01-1.2.3",
  "targets": ["robot-001"],
  "mode": "online",
  "schedule": null,
  "force": false,
  "created_at": "..."
}
```

板端落盘时同时保存：**Job + 已验签 Release**（或 content hash 引用）。

### 4.3 在线包 / 离线包

**online（瘦包）**：签名 Release + compose/config 等制品；**无**镜像 tar。  
**offline（胖包）**：上述 + `images/*-arm64.tar` / `*-amd64.tar`（按需）+ 各 tar 的 sha256。

多数发版不升 Agent：可不带 agent 载荷；V1.0–1.2 若本机 `< min_version` → **拒绝 Job**（不现场自升级）。

---

## 5. 业务 Release 生命周期（板端）（P0-1）

### 5.1 目录布局

```text
/opt/dr-robot/
├── releases/
│   ├── release-A/
│   │   ├── manifest.json          # 该 release 的节点切片 + 引用
│   │   ├── compose/
│   │   ├── configs/
│   │   └── release.env
│   ├── release-A-1/               # 更旧，可作 rollback
│   └── release-B/                 # staged candidate
├── active -> releases/release-A
├── rollback -> releases/release-A-1
└── staged -> releases/release-B   # 仅升级窗口存在
```

约定：

- **active**：当前已 COMMIT、正在跑的业务版本  
- **rollback**：上一成功版本，供失败回退  
- **staged**：已准备、未 COMMIT 的候选  

持久化数据目录（日志、业务 DB、地图等）**不得**放在 release 目录内。

### 5.2 本地记录（SQLite，见 §10）

```json
{
  "active_release": "release-A",
  "rollback_release": "release-A-1",
  "staged_release": "release-B",
  "active_before": "release-A"
}
```

### 5.3 回退目标来源（硬规则）

> **回退目标必须来自板端本地 `active_before` / `rollback` 指针，不得依赖 Job/Release 里服务端填写的 `previous_app_version`。**

服务端可记录历史版本供展示，但**设备裁决以本地为准**。服务端记录可能过期或与实机不一致。

COMMIT 成功时：

1. `rollback` ← 旧 `active`  
2. `active` ← 原 `staged`  
3. 清除 `staged`  
4. 更新本地 DB  

---

## 6. Compose 切换语义（P1-2）

必须规定：

1. **固定 `COMPOSE_PROJECT_NAME`**（写在 Release 节点上），禁止依赖目录名自动生成 project，避免换 release 目录冒出另一套容器  
2. 新 release 先 `docker compose config` 校验  
3. 所有镜像在 ACTIVATE 前拉取/load 完成并校验 digest  
4. 激活阶段使用 **`--pull never`**  
5. 停止顺序、启动顺序由 Release **显式给出**  
6. trial 阶段不允许 Docker/重启策略绕过 Agent 自动把旧栈拉起来造成混乱；由 Agent 控制启停  
7. relative volume / env 路径必须指向**本 release** 内路径  
8. 持久化数据在 release 外的固定数据目录  

示例（Q01 类，最终以实机依赖为准写入 Release）：

- 停止：`driver → system`  
- 启动：`system → driver`  

---

## 7. 双板事务协议（P0-2）

单板使用同一四阶段，无 peer 步骤。双板必须完整。

### 7.1 四阶段

```text
1. PREPARE
   下载/校验制品与镜像、Compose 展开、空间检查、写 staged
   所有参与节点 → PREPARED

2. ACTIVATE（仅在安全门禁通过后）
   按序停旧启新（candidate）
   所有参与节点 → ACTIVATED（进入 trial）

3. VERIFY
   各节点健康检查；NX 汇总

4. COMMIT 或 ROLLBACK
   全部健康 → COMMIT（切 active/rollback 指针）
   任一失败 → 所有参与节点 ROLLBACK
```

混合版本窗口**只允许存在于 trial**，且必须有 deadline；超时必须收敛到一致旧版或 `RECOVERY_REQUIRED`。

### 7.2 Trial lease（x86 必有，NX 建议亦有）

```text
trial_deadline = Release 规定（如 600s）
```

- 节点进入 ACTIVATED/trial 后启动 lease  
- **deadline 内未收到 NX 有效 COMMIT** → **自动 ROLLBACK 到本地 active_before**  
- 防止 NX 与 x86 失联后永久停留不同版本  

### 7.3 命令与幂等

- 每个命令携带：`job_id` + `transaction_epoch` + `nonce` + 有效期  
- 重复命令必须**幂等**  
- 旧 epoch 命令必须**拒绝**  
- NX 下发的 commit/rollback **可重试**  
- **每个节点独立持久化**本节点事务状态  

### 7.4 异常收敛（必须回答）

| 场景 | 行为 |
|------|------|
| NX 已切 B，x86 仍在 A 时断网 | x86 lease 超时自行回 A；NX 走 ROLLBACK 回 A；最终一致旧版 |
| x86 已起 B，NX Agent 重启 | NX 从 SQLite 恢复 epoch/phase，重发 VERIFY/COMMIT 或 ROLLBACK |
| NX 要回滚但连不上 x86 | NX 本机回滚；x86 靠 lease 自回滚；持续重试通知 |
| 健康失败 | 参与节点全部 ROLLBACK |
| 回滚也失败 | 进入 **RECOVERY_REQUIRED**，保持执行器安全态，等待人工/救援 |

---

## 8. 机器人升级安全门禁（P0-3）

下载/PREPARE 可在非安全态进行；**ACTIVATE 前必须通过门禁**。

### 8.1 PrepareForOTA 接口（业务状态机 → Agent）

```json
{
  "allowed": true,
  "safe_token": "xxx",
  "expires_at": "...",
  "robot_state": "idle",
  "actuator_safe": true,
  "battery_percent": 78,
  "fault_blocking": false
}
```

至少要求：

- `robot_state == idle`（或等价）  
- 无任务运行  
- 机器人静止  
- 执行器安全态  
- 电量高于阈值或正在充电  
- 无禁止升级的关键故障  
- `safe_token` 未过期  

**状态机不可达 → 拒绝激活，不得默认允许。**  
等待门禁时对外 phase = `WAITING_FOR_SAFE_STATE`，**禁止伪造进度增长**。

Agent 保持独立执行与回退能力；门禁只解决「能不能停机切栈」。

---

## 9. 健康检查契约（P0-4）

容器 Running ≠ 可 COMMIT。

```yaml
healthcheck:
  startup_timeout_sec: 60
  observation_sec: 120
  checks:
    - type: container
      service: iceoryx
      required: true
      severity: critical
    - type: container
      service: system_manager
      max_restart_count: 0
      required: true
      severity: critical
    - type: http
      url: http://127.0.0.1:8080/health
      required: true
      severity: critical
    - type: command
      service: servo-driver
      command: /opt/dr/bin/driver-health
      required: true
      severity: critical
    - type: ros
      topic: /driver/servo/status
      max_age_sec: 3
      required: true
      severity: critical
```

约定：

- **critical**：失败立即进入回滚  
- **optional**：告警，不单独阻断（策略可配）  
- **observation_sec**：持续观察，避免刚启动就 COMMIT  
- 关注：容器重启次数、HTTP/命令、ROS topic 新鲜度、硬件通信、严重故障位  

正式 OTA 前须为关键服务补齐健康契约（不能仅有 Web healthcheck）。

---

## 10. 持久化与断电恢复（P0-5）

### 10.1 推荐：SQLite WAL

库内至少：

- `jobs` / `job_nodes`  
- `releases` / `components`  
- `events` / `artifacts`  

状态转换：

```text
BEGIN
  记录 intent
COMMIT + fsync
执行外部操作（docker 等）
记录 result（再事务提交）
```

### 10.2 若短期用 JSON，必须原子写

```text
写 state.tmp
→ fsync(state.tmp)
→ rename(state.tmp, state.json)
→ fsync(父目录)
```

并配备 append-only journal 或备份。仅写「阶段更新都写盘」**不足以**保证断电安全。

### 10.3 恢复目标

断电重启后只能：

- 回到**旧稳定 active**，或  
- 完成**已 COMMIT** 的新版本  

不允许停在半切换且无 lease/无恢复路径的中间态。

---

## 11. Agent 自升级策略（P0-6）

### 11.1 V1.0–1.2（本基线默认）

- **不做 Agent OTA 自升级**  
- 出厂/装机预装满足协议的 Agent  
- Release 声明 `agent.min_version`；本机不满足 → **拒绝 Job**，业务不切换  

### 11.2 V1.3：独立 bootstrap（若做自升级）

新 Agent 若起不来，**不能**指望它自己执行「回 prev」。

```text
ota-bootstrap.service   # 极小、不随普通业务 OTA 更新
├── 启动 current agent
├── 监控启动确认文件
├── 超时或连续失败
├── 切回 previous agent
└── 再启动旧 Agent
```

```text
/usr/local/dr-ota/
├── bootstrap/ota-bootstrap
├── releases/1.3.0/...
├── releases/1.4.0/...
├── current
└── previous
```

Agent 不得成为自己唯一的回退执行者。

---

## 12. 磁盘与镜像清理（P0-7）

### 12.1 受保护集合

- active release  
- rollback release  
- 当前 Job 的 staged release  
- 正在运行容器引用的镜像  
- Agent / recovery / bootstrap 所需资源  

### 12.2 清理顺序

1. 失败的 staging  
2. 过期下载缓存  
3. dangling image  
4. 不被任何保留 release 引用的镜像  
5. 若仍不足，且 active 已稳定超过保护期 → 可删**更老的** rollback（保留「当前 active 作为下一 candidate 的回退」所需的那份）  
6. 保留 active，再下载 candidate  
7. 若仍放不下 `active + candidate` → **拒绝升级**  

符合：`active=A, rollback=A-1, candidate=B` 时，空间不足可删更老历史，但**不能删当前 A**；B 失败时仍能回 A。

镜像与 release 建立引用关系；**由 OTA Agent 统一按引用 GC**。部署脚本中的「自动乱删旧镜像」应取消。

下载前根据 Release 的 size 字段做空间预检。

---

## 13. 安全设计（P0-8）

1. **不要对可重排 YAML 做语义签名**；用 canonical JSON/CBOR，或对原始 Manifest 字节签名且禁止「解析再序列化再验签」  
2. `manifest.sig` 外置  
3. 签名覆盖 release 关键字段与全部 artifact/digest（见 §4.1）  
4. **x86 必须自己验证厂商 Release 签名**，不能只信「NX 转发的无签名任务」  
5. NX ↔ x86：**mTLS**  
6. NX 下发 = 已签名 Release + 设备任务上下文（job_id/epoch/nonce/有效期）  
7. **NX 不持有厂商发布私钥**  
8. 型号 / 架构 / digest / hash 任一不匹配 → 业务完全不切换  
9. **防回放，不禁止人工降级。** Release 带单调 `sequence`。普通 Job 只接受 `sequence` 大于本机已提交版本的包；把旧包原样再推一遍必须拒绝。回到旧业务版本必须是一条新的、已签名的降级 Job（`intent: downgrade`），目标可以是旧 `release_id`，但 Job 本身是新的。`force` / `downgrade` 只给这次人工操作，不能当成日常升级开关。自动失败回滚仍走本地 `active_before`，不依赖重放旧包。

---

## 14. 状态机（P1-4）

```text
RECEIVED
→ PREFLIGHT
→ DOWNLOADING
→ VERIFYING          # 制品/镜像校验
→ STAGED
→ WAITING_FOR_SAFE_STATE
→ PREPARING_NODES    # 事务 PREPARE 确认
→ ACTIVATING
→ VERIFYING_TRIAL
→ COMMITTING
→ SUCCEEDED
```

失败分支：

```text
激活前失败 → FAILED（业务不动）
激活后失败 → ROLLING_BACK → ROLLED_BACK | ROLLBACK_FAILED
回滚失败   → RECOVERY_REQUIRED
```

另需定义：用户取消、Job 超时、重试次数、rollback 后是否允许重试同一 release、`RECOVERY_REQUIRED` 的安全态要求。

---

## 15. 进度模型（P1-3）

下载与升级拆开；回退单独；总进度仅简略展示。

```json
{
  "job_id": "job-10027",
  "release_id": "q01-1.2.3",
  "phase": "DOWNLOADING",
  "download": {
    "percent": 72,
    "bytes_done": 123,
    "bytes_total": 456,
    "component": "q01-system"
  },
  "upgrade": {
    "percent": 0,
    "phase": "NOT_STARTED"
  },
  "rollback": null,
  "overall_percent": 45,
  "elapsed_sec": 185,
  "started_at": "...",
  "updated_at": "...",
  "scope": ["nx"],
  "message": "正在下载镜像",
  "error": null
}
```

- 下载：真实字节（Docker API pull 进度 / 文件拷贝）  
- 升级：步骤进度  
- `WAITING_FOR_SAFE_STATE`：**不伪造百分比增长**  
- 上报：x86 → NX 聚合 → Server/UI；下载进度节流  

---

## 16. 升级主流程（汇总）

### 16.1 在线（NX 单板 / 双板同一骨架）

1. 收 Job → 拉/取 Release → 验签 → **SQLite 落盘**  
2. PREFLIGHT：机型、agent min、空间、（双板）NAT/DNS/时间/CA/registry 鉴权  
3. DOWNLOADING：参与节点按 digest 直拉（x86 经 NX NAT）；进度上报。断网失败**保留本地未完成文件**，下次同一 Job / 同一 digest 续传，不整包重下。Docker 层缓存同样保留；校验通过前不切换业务。  
4. VERIFYING → 展开到 `staged`  
5. WAITING_FOR_SAFE_STATE → PrepareForOTA  
6. PREPARING_NODES → ACTIVATE（`--pull never`，按 stop/start order）  
7. VERIFYING_TRIAL（observation 窗口）  
8. 全过 COMMIT；否则 ROLLBACK；lease 兜底  

### 16.2 离线（V1.2）

offline 包进 NX inbox → 验签 → load arm64 →（若有）传 amd64 tar 给 x86 → digest 校验 → 其后同事务流程。

### 16.3 网络预检与限流（双板，V1.1+）

- NX NAT 状态、x86 DNS/时间/CA、Registry 鉴权预检  
- 下载并发与带宽限制，避免拖垮业务  
- 不必永远并行拉取，可按链路能力串行/限流  

---

## 17. 公有云 / 私有化 / 离线

| 形态 | Server | 镜像 | 机器人 |
|------|--------|------|--------|
| 公有云 | 托管 | online 直拉 | NX 配云端 |
| 私有化 | 客户内网 | offline 导入 Harbor 后 online | 改 endpoint/证书 |
| 纯隔离 | 可无 | 仅 offline | NX 扫 inbox |

---

## 18. Agent / 业务 / 协议版本

| 对象 | 版本 |
|------|------|
| 业务 | `release_id` / app 版本 |
| Agent | `agent_version`（非每次升） |
| 协议 | `protocol_version` |

- 用 `protocol_version` + `agent.min_version` 约束，不绑死同号  
- 新 agent 尽量兼容旧协议/旧业务，便于只回滚业务  
- 不兼容则 bump 协议并抬高 min；V1.3 前靠装机升级 Agent  

状态机等业务容器随 **release 整包**升/回，不靠 agent 版本对齐业务内部协议。

---

## 19. 实施节奏（调整后）

### V1.0：NX 单板在线 OTA

- 签名 Release Manifest + Job 分离  
- Compose/config 瘦包与切换语义  
- Docker digest 拉取  
- 下载/升级进度分离  
- active / rollback / staged  
- 空间清理  
- 健康检查契约  
- 自动回退（本地指针）  
- SQLite Job 状态 + 断电恢复  
- 机器人安全门禁  

### V1.1：NX+x86 双板事务

- worker mTLS  
- prepare/activate/verify/commit  
- trial lease  
- 整 Job 回退与进度汇总  
- 断网/分区恢复  
- 有运控机型两板同一业务 release，拒绝只含一块板的包  
- NAT 预检与限流  

### V1.2：离线胖包

- arm64/amd64 归档 + sha256  
- U 盘/inbox  
- x86 文件传输进度  
- load 后 digest 验证  

### V1.3：Agent 自升级

- 不可变 bootstrap  
- current/previous  
- 启动确认与超时回退  
- Agent 与 Job 协议兼容恢复  

MCU/内核/OS：仅 Executor 占位。

---

## 20. 生产验收标准（方案定稿门槛）

1. 下载阶段任意断网，当前业务不受影响；已下载内容保留，恢复网络后按 digest 续传。  
2. NX 在任意状态点断电，启动后只能回到旧稳定版本或完成已提交的新版本。  
3. x86 在 trial 阶段与 NX 失联，超时自动回退。  
4. NX/x86 任一节点失败，最终两板回到一致 release。  
5. 新版本健康检查失败，旧版本无需联网即可恢复。  
6. rollback 镜像绝不被普通 GC 删除。  
7. 空间不足时可删除更老 rollback，但不能删除当前 active。  
8. 签名、hash、型号、架构或 digest 不匹配时，业务完全不切换。  
9. 相同 Job 重复下发不会重复执行有副作用的步骤。  
10. 下载、升级、回退进度分别可观察。  
11. 业务状态机不允许升级时，Agent 不得强制切换。  
12. 新 Agent 无法启动时，由外部 bootstrap 自动恢复旧 Agent（V1.3）。  
13. 双板网络分区后不存在永久混合版本。  
14. 回退也失败时进入明确的 `RECOVERY_REQUIRED` 安全状态。  

---

## 21. 一句话总结

> **以 NX 为控制面；有运控时机型业务层两块板必须一起升。四层分开：业务切容器、Agent 只重启进程、MCU 复位该芯片、系统层才重启板卡。普通升级拒绝重放旧包，人工降级走新的签名 Job。下载失败保留本地文件并续传。Release 与 Job 分离；active/rollback/staged；门禁、健康检查、trial lease、SQLite 与按引用清理。当前为架构基线。**

---

## 22. 建议开工接口清单

1. Release Manifest / Job Document schema 与签名规范  
2. 板端 release 目录与 SQLite schema  
3. PrepareForOTA 门禁 API  
4. 健康检查契约 schema  
5. 单板状态机 + 进度模型  
6. Compose 切换执行器（project 名、顺序、`--pull never`）  
7. 清理与空间预检  
8. （V1.1）双板事务命令、epoch、lease、mTLS  
9. （V1.2）offline 包格式  
10. （V1.3）ota-bootstrap  

---

## 附录 A. 相对初稿已关闭的关键缺口

| 编号 | 主题 | 基线结论 |
|------|------|----------|
| P0-1 | 业务版本板端保存 | active/rollback/staged + 本地指针回退 |
| P0-2 | 双板整 Job 回退 | 四阶段事务 + trial lease + epoch |
| P0-3 | 安全门禁 | PrepareForOTA + WAITING_FOR_SAFE_STATE |
| P0-4 | 健康检查 | 多层 critical/optional + observation |
| P0-5 | 断电一致性 | SQLite WAL（或原子 JSON） |
| P0-6 | Agent 自升级风险 | V1.0–1.2 不做；V1.3 bootstrap |
| P0-7 | 磁盘镜像清理 | 受保护集合 + 清理顺序 |
| P0-8 | 安全细节 | canonical 签名、x86 自验签、mTLS、NX 无发布私钥 |
| P1 | Release/Job 拆分、Compose 语义、进度拆分、状态机、artifact、限流 | 已纳入 |
| 节奏 | 实施顺序 | V1.0 单板 → 1.1 双板 → 1.2 离线 → 1.3 Agent |
| 验收 | 14 条 | §20 |
