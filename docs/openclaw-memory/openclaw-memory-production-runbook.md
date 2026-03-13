# OpenClaw 线上记忆系统生产化改造方案

GitHub 文档地址：
- https://github.com/583721/openclaw-deployable-package-clean/blob/main/docs/openclaw-memory/openclaw-memory-production-runbook.md
- https://github.com/583721/openclaw-deployable-package-clean/blob/main/docs/openclaw-memory/openclaw-memory-merged-architecture.md
- https://github.com/583721/openclaw-deployable-package-clean/blob/main/docs/openclaw-memory/openclaw-memory-deploy-prompt.txt

适用对象：

- 你当前这台 Ubuntu 24.04 的线上 OpenClaw
- 由其他 AI 直接代你部署
- 目标是比视频里的“纯 LanceDB 长期记忆插件”更稳、更适合生产

---

## 一句话结论

不要直接把视频里的第三方长期记忆插件当作线上主方案。

更稳的生产方案是：

1. 保持 `plugins.slots.memory = "memory-core"`
2. 把 `memory.backend` 切到官方 `qmd`
3. 用 `MEMORY.md + memory/YYYY-MM-DD.md` 作为唯一真相源
4. 开启 `memoryFlush`
5. 开启 `QMD` 的混合检索和 session 索引
6. 把“自动学习”做成**受控写入**，不要一上来全自动 auto-capture

这套方案比视频更适合线上，原因是：

- 官方能力，兼容性更稳
- Markdown 是真相源，容易审计、备份、回滚
- QMD 自带 `BM25 + vector + rerank` 思路，已经覆盖视频里最值钱的部分
- 不会因为切换到 `memory-lancedb` 失去官方 `memory_search / memory_get`
- 出问题时可以直接查看和编辑 `MEMORY.md`

---

## 最终目标架构

生产上采用三层记忆：

### 第 1 层：显式长期记忆

- 文件：`MEMORY.md`
- 用途：项目铁律、运维经验、稳定偏好、已确认的长期事实
- 特点：人工可读、可审计、可回滚

### 第 2 层：日常工作记忆

- 文件：`memory/YYYY-MM-DD.md`
- 用途：当天会话、调试过程、临时上下文
- 特点：每天滚动记录，后续可以晋升到 `MEMORY.md`

### 第 3 层：检索层

- 后端：官方 `QMD`
- 用途：让 `memory_search` 支持更好的混合检索
- 内容来源：
  - `MEMORY.md`
  - `memory/**/*.md`
  - 可选的 session transcript

---

## 为什么这套比视频方案更适合线上

视频里的方向是对的，但更像“增强记忆插件产品化”，不够像“线上治理方案”。

线上最关键的不是“能不能记住”，而是：

- 记住什么
- 谁能召回
- 什么时候写入
- 误记了怎么删
- 召回错了怎么回滚

所以我建议：

### 1. 不把第三方向量库当真相源

真相源必须是 Markdown：

- `MEMORY.md`
- `memory/YYYY-MM-DD.md`

向量库只做索引和召回，不做唯一事实库。

### 2. 不一上来就全自动学习

先做：

- 自动召回
- 受控写入

后做：

- 自动提炼
- 自动晋升为长期规则

### 3. 不直接切 `memory-lancedb`

原因：

- 官方 `memory-core` 提供 `memory_search / memory_get`
- 官方文档已经明确支持 `memory.backend = "qmd"`
- 切去 `memory-lancedb` 会偏离当前官方主线

### 4. 用 QMD 覆盖视频里最核心的优势

视频里最值钱的其实不是 “LanceDB” 三个字，而是：

- 混合检索
- rerank
- 长期记忆召回
- 更好的中文/配置/运维内容命中

官方 `QMD` 已经很接近这个方向。

---

## 推荐落地顺序

严格按下面 4 个阶段做，不要跳步。

### Phase 0：先把 Gateway 服务化

当前线上如果还是手工跑进程，先改成标准服务。

目标：

- `openclaw gateway status` 可用
- `openclaw gateway restart` 可用
- 重启后配置可自动生效

### Phase 1：先上官方记忆底座

做这些：

- 保持 `plugins.slots.memory = "memory-core"`
- 开启 `memoryFlush`
- 建立 `MEMORY.md` 模板
- 建立 `memory/` 日志目录

目标：

- 模型开始把长期规则写到文件
- 还没有引入新的复杂插件风险

### Phase 2：把检索升级到 QMD

做这些：

- 安装 QMD
- 设置 `memory.backend = "qmd"`
- 开启 session transcript 索引
- 控制召回数量和注入长度

目标：

- 提升混合检索效果
- 让历史问题、运维规则、配置经验能被更稳定召回

### Phase 3：再加“受控自动学习”

做这些：

- 只允许把成功闭环后的规则晋升到 `MEMORY.md`
- 先人工/半自动，再考虑全自动

目标：

- 避免噪声记忆和错误经验污染

### Phase 4：最后才考虑第三方长期记忆插件

只在下面条件都满足时考虑：

- 官方 `memory-core + qmd` 已稳定跑 1-2 周
- 召回命中率和质量可接受
- 已有清晰的删除/修订流程
- 已有用户/项目隔离策略

如果要试，也先上测试环境，不要直接上主线上。

---

## 推荐的生产方案（最终版）

### 方案名

**官方 Memory Core + QMD 混合检索 + 受控记忆写入**

### 必须遵守的原则

1. 记忆源文件优先，向量库只做索引
2. 默认不自动把所有对话写入长期记忆
3. 任何长期规则都必须可审计、可删除、可回滚
4. 检索召回必须限流，不能无限塞上下文
5. 任何涉及密码、token、cookie、私钥的内容禁止写入长期记忆

---

## 其他 AI 需要实际做的事情

下面这段可以原样执行。

### A. 备份

先备份：

- `~/.openclaw/openclaw.json`
- `~/.openclaw/`
- 当前 OpenClaw 版本号
- 当前 Gateway 服务状态

### B. 规范化服务

如果 Gateway 还不是标准服务，先修成标准服务。

验收标准：

- `openclaw gateway status` 成功
- `openclaw gateway restart` 成功

### C. 保持 memory 插槽为官方 core

要求：

```json5
{
  plugins: {
    slots: {
      memory: "memory-core"
    }
  }
}
```

不要切到：

- `memory-lancedb`
- 第三方 memory 插件

除非是测试环境。

### D. 开启 memory flush

在配置里加入或校正：

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write lasting rules and verified facts to MEMORY.md or memory/YYYY-MM-DD.md. Reply with NO_REPLY if nothing should be stored."
        }
      }
    }
  }
}
```

### E. 切到 QMD 后端

配置目标：

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      includeDefaultMemory: true,
      sessions: {
        enabled: true,
        retentionDays: 30
      },
      update: {
        onBoot: true,
        waitForBootSync: false
      },
      limits: {
        maxResults: 8,
        maxSnippetChars: 700,
        maxInjectedChars: 2200,
        timeoutMs: 5000
      }
    }
  }
}
```

注意：

- 字段名如果和当前官方版本略有差异，必须以当前官方文档和本机 schema 为准
- 但语义目标不要改

### F. 安装 QMD

按当前官方文档执行。

文档里提到的方向是：

- 安装 `qmd`
- 确保 QMD 在 Gateway 的 PATH 中可用
- 首次运行时允许模型和索引初始化

### G. 初始化记忆文件

如果不存在，则创建：

- `~/...workspace.../MEMORY.md`
- `~/...workspace.../memory/`

`MEMORY.md` 模板建议固定为：

```md
# Long-Term Memory

## Project Facts

## Operational Rules

## Infra / Deployment Notes

## Known Incidents

## User Preferences
```

### H. 加一条“记忆写入纪律”

把下面规则补到主 agent 的系统提示或团队开发规则里：

- 只有“长期有效”的信息才写入 `MEMORY.md`
- 当天临时过程写入 `memory/YYYY-MM-DD.md`
- 禁止把密码、token、cookie、私钥、SSH 凭据写入记忆
- 修复类经验必须写成“问题 + 原因 + 处理 + 避免方式”
- 在重试失败任务前，先做一次 `memory_search`

### I. 开启 session transcript 索引

目的：

- 让最近历史在检索层可查
- 但不等于把全部 session 自动提升为长期记忆

### J. 做 5 个验收测试

必须全部通过：

1. 写入 `MEMORY.md` 的一条运维规则，重开新 session 后可被 `memory_search` 找到
2. 写入当天 `memory/YYYY-MM-DD.md` 的一条临时记录，能被召回
3. 插入一条包含 token 的文本，确认不会进入长期记忆
4. 故意让 QMD 不可用，`memory_search` 仍能平稳降级，不影响聊天
5. `openclaw gateway restart` 后配置仍生效

### K. 做回滚方案

回滚必须简单：

1. 恢复 `openclaw.json` 备份
2. 把 `memory.backend` 改回默认
3. 重启 Gateway
4. 保留 `MEMORY.md` 和 `memory/*.md`
5. 不删除旧数据，除非人工确认

---

## 我建议额外加进去的“增强版改进”

这是比视频方案更适合线上 OpenClaw 的部分。

### 改进 1：不要自动把原始聊天直接长期化

正确做法：

- 原始聊天 -> `memory/YYYY-MM-DD.md`
- 成熟规则 -> `MEMORY.md`

### 改进 2：给记忆内容固定格式

推荐写成：

```md
- 问题：
- 原因：
- 处理：
- 避免方式：
- 适用范围：
```

这样比自然语言大段描述更利于检索和复用。

### 改进 3：把“失败经验”作为重点资产

视频里这部分方向很好，建议保留并强化：

- 修复过的坑
- 部署错误
- 配置陷阱
- 误判路径

这些内容比“普通偏好”更值钱。

### 改进 4：先 recall 再 retry

把这条写进 agent 纪律：

- 当任务失败且属于已做过的工程领域时，先 `memory_search`
- 找到相关经验后再重试

### 改进 5：禁止记住敏感信息

必须额外加规则：

- password
- token
- apiKey
- cookie
- ssh key
- private key
- session secret

全部禁止长期存储。

### 改进 6：把“用户偏好”和“项目规则”分开

不要混写。

推荐：

- `User Preferences`
- `Project Facts`
- `Operational Rules`

分栏后召回更稳定。

### 改进 7：先不用第三方 autoCapture

原因：

- 噪声太多
- 不容易知道为什么写入
- 容易污染长期记忆

先用：

- 官方 `memoryFlush`
- 明确的写入规则
- 受控召回

### 改进 8：以后如果真要接视频里的插件，只放测试环境

测试环境验证项：

- recall 命中率
- 错误召回率
- 重复写入率
- 删除/忘记流程是否可用
- 重启/升级后的兼容性

---

## 不推荐的做法

不要让其他 AI 这样做：

1. 直接把线上 memory 插槽切到未知第三方插件
2. 不看 schema 就乱写配置字段
3. 把原始聊天全文直接长期存库
4. 不做敏感信息过滤
5. 不做备份直接改 `openclaw.json`
6. 不验证回滚就上线

---

## 给其他 AI 的执行提示词

把下面这段直接发给其他 AI：

```text
请在 Ubuntu 24.04 的线上 OpenClaw 上实施“官方 Memory Core + QMD 混合检索 + 受控记忆写入”方案。

硬性要求：
1. 不要切换到第三方 memory 插件作为主方案
2. 保持 plugins.slots.memory = "memory-core"
3. 开启 agents.defaults.compaction.memoryFlush
4. 把 memory.backend 切到 qmd
5. 启用默认 memory 文件索引
6. 启用 session transcript 索引，保留 30 天
7. 限制 recall 注入长度和结果数量
8. 初始化 MEMORY.md 模板
9. 添加记忆写入纪律：禁止密码/token/cookie/私钥进入长期记忆
10. 备份、验收、回滚方案必须一起完成

请按以下顺序执行：

A. 备份当前 OpenClaw 配置和状态目录
B. 确认 Gateway 已服务化，可 restart/status
C. 审核当前 OpenClaw 版本与 memory/qmd 配置 schema
D. 安装并验证 qmd 可执行
E. 修改 openclaw.json
F. 初始化 MEMORY.md 与 memory/ 目录
G. 重启 Gateway
H. 执行 5 个验收测试
I. 输出最终报告：改了什么、实际配置片段、验证结果、回滚方式

注意：
- 任何具体字段名必须以当前本机 OpenClaw 版本的官方 schema 和 docs 为准
- 但语义目标不能偏离本 runbook
- 如果某字段在当前版本不存在，请找官方等价字段，不要自己发明
```

---

## 参考依据

这份方案主要基于以下可确认内容：

- 官方插件文档：`docs/tools/plugin.md`
- 官方记忆文档：`docs/concepts/memory.md`
- 官方 CLI 记忆文档：`docs/cli/memory.md`
- 官方内置插件源码：
  - `extensions/memory-core`
  - `extensions/memory-lancedb`

核心判断：

- 视频方案最值得借鉴的是“混合检索 + 长期经验沉淀”
- 但线上最稳的实施路径，应优先走 OpenClaw 官方主线能力，再把视频里的经验治理思想加进去
