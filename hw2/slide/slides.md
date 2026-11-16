---
marp: true
theme: default
paginate: true
size: 16:9
style: |
    section {
      font-family: 'Calibri', '华文中宋', 'Microsoft YaHei', sans-serif;
      background-color: #ffffff;
      padding: 64px 48px 34px 48px;
      font-size: 25px;
      line-height: 1.4;
      color: #1e293b;
      justify-content: flex-start;
    }
    section::after {
      position: absolute;
      right: 40px;
      bottom: 14px;
      font-size: 0.72em;
      color: #9e9e9e;
      font-family: 'Calibri', '等线', sans-serif;
    }
    section header {
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      margin: 0;
      padding: 10px 48px;
      background: #3949ab;
      color: white;
      font-family: 'Calibri', '等线', sans-serif;
      font-size: 0.8em;
      font-weight: bold;
      line-height: 1.65;
    }
    h1, h2 {
      font-family: 'Calibri', '等线', 'Microsoft YaHei', sans-serif;
      color: #3949ab;
      margin-top: 0.1em;
      margin-bottom: 0.35em;
    }
    h2 {
      font-size: 1.32em;
      border-bottom: 2px solid #e8eaf6;
      padding-bottom: 4px;
    }
    h3 {
      color: #3949ab;
      font-size: 0.96em;
      font-weight: bold;
      margin: 0.25em 0 0.2em 0;
    }
    ul, ol {
      margin: 0.2em 0;
      padding-left: 1.15em;
    }
    li {
      margin-bottom: 0.28em;
    }
    table {
      font-size: 0.76em;
      width: 100%;
      border-collapse: collapse;
      margin: 0.25em 0;
    }
    th {
      background-color: #e8eaf6;
      color: #283593;
    }
    td, th {
      padding: 6px 9px;
    }
    section.lead {
      text-align: center;
      background: #f5f7fa;
      padding-top: 0;
      justify-content: center;
    }
    section.lead header {
      display: none !important;
    }
    section.lead h1 {
      color: #3949ab;
      font-size: 2.0em;
      margin-top: 0;
    }
    section.lead h2 {
      color: #5c6bc0;
      border: none;
    }
    section.cover {
      text-align: left;
      padding: 80px 90px;
      background: #f5f7fa;
      justify-content: center;
    }
    section.cover header {
      display: none !important;
    }
    section.cover h1 {
      position: static;
      background: none;
      color: #3949ab;
      font-size: 1.85em;
      margin-bottom: 0.25em;
      padding: 0;
      line-height: 1.2;
      border: none;
    }
    section.cover h2 {
      color: #1e293b;
      font-family: 'Calibri', '霞鹜文楷', 'Microsoft YaHei', sans-serif;
      font-weight: normal;
      font-size: 1.2em;
      border: none;
      margin-bottom: 0.8em;
    }
    .diagram {
      display: block;
      margin: 0.35em auto 0;
      max-width: 98%;
      width: auto;
      height: auto;
      object-fit: contain;
      border: 1px solid #e2e8f0;
      border-radius: 6px;
    }
    .h-290 {
      max-height: 290px;
    }
    .h-320 {
      max-height: 320px;
    }
    .h-500 {
      max-height: 500px;
    }
    .h-520 {
      max-height: 520px;
    }
    .mockup {
      display: block;
      margin: 0.15em auto 0.3em;
      max-width: 100%;
      width: auto;
      height: auto;
      max-height: 330px;
      object-fit: contain;
      border: 1px solid #cbd5e1;
      border-radius: 6px;
      box-shadow: 0 2px 6px rgba(15, 23, 42, 0.08);
    }
    code {
      font-size: 0.86em;
      background: #f1f5f9;
      padding: 1px 4px;
      border-radius: 3px;
    }
    .blue {
      color: #3949ab;
      font-weight: bold;
    }
    .columns {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 0.9em;
      align-items: start;
    }
    .columns-wide-left {
      display: grid;
      grid-template-columns: 1.02fr 0.98fr;
      gap: 0.9em;
      align-items: start;
    }
    .card {
      background: #f8fafc;
      border-left: 4px solid #3949ab;
      padding: 12px 16px;
      border-radius: 4px;
      margin-bottom: 0.45em;
      font-size: 0.92em;
    }
    .card h3 {
      margin-top: 0;
      margin-bottom: 0.25em;
    }
    .compact {
      font-size: 0.92em;
    }
    .ui-col {
      font-size: 0.88em;
    }
math: mathjax
---

<!-- _class: cover -->

# A Declarative Personal SysOps Agent<br/>with Adaptive Long-Term Memory

## System Analysis & Design — Assignment 2 设计汇报

2450313 徐佳咏 &ensp; 2351428 徐之涵 &ensp; 2352626 马小云 &ensp; 2451405 刘宸瑞

---

<!-- header: 一、架构设计 -->

## 1.1 逻辑架构

采用模块化单体架构，将控制平面划分为 4 个子系统：

<img class="diagram h-500" src="assets/logical_arch.png" alt="逻辑架构图">

---

## 1.2 部署架构

Web 控制台、控制面单二进制文件、目标节点无入侵

<img class="diagram h-500" src="assets/deployment_arch.png" alt="部署架构图">

---

## 1.3 技术架构与竞品对比

<div class="columns">
<div>

### 竞品对比与定位

| 对比对象        | 既有局限                  | 我们的定位                |
| :-------------- | :------------------------ | :------------------------ |
| **Ansible**     | playbook 完全静态，无推理 | 静态声明 + 大模型动态诊断 |
| **Datadog**     | 企业级部署过重            | 单二进制个人轻量部署      |
| **Claude Code** | Coding 特化，非多节点运维 | 多节点运维工具            |

<div class="card" style="margin-top: 0.45em;">

弥合 重型企业平台 与 手工 SSH 排障 之间的断层

</div>

</div>
<div>

### 技术栈选型与权衡

| 层级         | 技术                   | 理由                       |
| :----------- | :--------------------- | :------------------------- |
| **后端传输** | **Go + 原生 SSH**      | 控制机单二进制文件，无依赖 |
| **存储检索** | **SQLite FTS5**        | 运维成本低，兼顾人类理解   |
| **前端通信** | **Vue 3 + SSE**        | 自动重连，单向推流         |
| **安全解析** | **Shell AST + 大模型** | 防止管道与子命令绕过       |

</div>
</div>

---

<!-- header: 二、关键设计决策与权衡 -->

## 2.1 并发控制与自适应记忆

<div class="columns">
<div class="card">

### 1. 单二进制 + 节点锁

- **痛点**：防止同节点巡检、诊断与修复并发冲突
- **放弃**：微服务分布式锁、无锁自由并发
- **决策**：采用节点锁，同漂移合并、异类排队，用串行化换取零写竞争

</div>
<div class="card">

### 2. SQLite FTS5 + 三分支演进

- **痛点**：需复用历史方案
- **放弃**：PostgreSQL + pgvector、Milvus
- **决策**：使用文本关键词搜索匹配，仅召回匹配的活跃经验，验证后按“新建 / 命中 / 修订”演进；效果上弱于向量语义，但明显减轻部署与运维成本，且适用于需频繁更新、人工修订的场景

</div>
</div>

---

## 2.2 事务恢复与实时通信

<div class="columns">
<div class="card">

### 3. 分步补偿

- **痛点**：SSH 多步执行非原子，中途失败会破坏现场
- **放弃**：人工善后脚本、目标机快照或驻留 Agent
- **决策**：每个写操作强绑定回滚命令，失败即中止并进入部分失败态；代价是增加分析与审查耗时

</div>
<div class="card">

### 4. 前端事件流

- **痛点**：长耗时操作需实时刷新，且断线不能重复执行
- **放弃**：高频轮询、全双工 WebSocket
- **决策**：服务端单向 SSE 推流，客户端自动重连；代价是客户端需维护历史窗口与过期全量同步

</div>
</div>

---

<!-- header: 三、核心专项设计与模型 -->

## 3.1 Agentic 工作流设计

<div class="columns-wide-left">
<div>

<img class="diagram h-520" src="assets/agent_orch.png" alt="Agent内部编排与工具调用图">

</div>
<div class="compact">

- **角色分离**：诊断 Agent 仅推理提案；审查 Agent 只做危险分级，不接触执行器
- **危险指令拦截**：本地 AST 解析 + 独立审查 Agent
- **脱敏**：敏感凭据本地脱敏后再发送给大模型 API
- **熔断**：诊断运行轮次与总时长限制，超限自动转人工

</div>
</div>

---

## 3.2 状态机与恢复闭环

<div class="columns">
<div>

<img class="diagram h-520" src="assets/incident_sm.png" alt="Incident生命周期状态机">

</div>
<div class="compact">

1. **巡检排重**：零模型成本发现漂移，并发信号自动合并或排队
2. **有界诊断**：结合历史记忆只读排查，输出带回滚契约的提案
3. **审批与补偿**：人工放行后独占执行；任一步失败立即中止，支持逆序回滚
4. **复测与蒸馏**：探针确认恢复后沉淀记忆

</div>
</div>

---

## 3.3 UI/UX 页面功能展示

<div class="columns ui-col">
<div>

<img class="mockup" src="../mockups/screen1.png" alt="节点总览界面">

### 节点总览与接入

- 分离展示 SSH 连接、健康与执行锁状态
- 支持私钥录入、一键测试等

</div>
<div>

<img class="mockup" src="../mockups/screen2.png" alt="策略编写与漂移对比界面">

### 声明式策略与漂移检查

- 左侧编辑 YAML 草稿，右侧实时对比期望与实测差异
- 节点锁定时自动禁用策略激活与手动探测

</div>
</div>

---

## 3.3 UI/UX 页面功能展示

<div class="columns ui-col">
<div>

<img class="mockup" src="../mockups/screen3.png" alt="诊断审批与部分失败恢复台">

### 诊断审批与失败恢复台

- 聚合脱敏排查证据、根因摘要与每步备份/回滚命令
- 中途失败时清晰标出各步状态，支持一键触发逆序回滚

</div>
<div>

<img class="mockup" src="../mockups/screen4.png" alt="自适应记忆中心">

### 自适应记忆

- 支持按故障关键词检索历史经验
- 保留完整版本，支持人工修订迭代或归档

</div>
</div>

---

<!-- _class: lead -->

# Thank You
