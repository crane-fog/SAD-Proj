<style>
.markdown-preview.markdown-preview {
    font-family: "Times New Roman", "宋体";
}
</style>

## Idea

---

> **AI Tools Reflection**
>
> - Tool chain: Gemini 3.8 Flash in Google AI Studio
> - Cost: covered by Google AI Pro subscription
> - Pros / cons: communicate about the project concept, helpful in brain storming and English translations, but easily provides ideas that are too complex or too simple
> - Verification: manually review & rewrite
> - Enhancement: correct & rewrite AI output to align with our true design intent
> - Artifacts: no specific artifacts

---

### 1. Identity

- **Title**: A Declarative Personal SysOps Agent with Adaptive Long-Term Memory
- **Team Name**: Moonlight
- **Members**: 2450313 徐佳咏、2351428 徐之涵、2352626 马小云、2451405 刘宸瑞

---

### 2. Problem and Motivation

- **Background**:
  In the era of cloud-native development, independent developers, homelab enthusiasts, and academic researchers may manage from a few to dozens of cloud instances, local workstations, and containerized services.
  However, traditional operational tools suffers from extreme polarization: enterprise-grade tools are overly complex and heavily configured, making them not suitable for individual-scale use; meanwhile, manual operations are also unsuitable for handling day-to-day operational toil.
  For example, when encountering the exact same container OOM kill, failed SSL certificate renewal, or firewall block months later, developers have often long forgotten their previous troubleshooting steps and temporary scripts, needlessly wasting time and effort.
- **Main Goals**:
  To construct a lightweight, single-daemon SysOps assistant for personal developers that can perform declarative agentic state reconciliation, and retains adaptive long-term memory of past incidents.
- **Scope**:
  Focused on individual-scale development.

---

### 3. Solution

- **Core Functionality**:
  The project provides a Web control panel, adaptive long-term memory, and a declarative agentic state reconciliation engine for personal developers to manage their small fleets of servers and containers.
- **Key Characteristics**:
    - declarative agentic state reconciliation
    - adaptive long-term memory
- **Intended Users**: Independent developers, homelab enthusiasts, and CS related academic researchers.
- **Usability Goals**: Lightweight deployment and easy configuration.

---

### 4. Market Context

| Similar Products | Utilities & Limitations                                                       | Our Novelty                                                                     |
| :--------------- | :---------------------------------------------------------------------------- | :------------------------------------------------------------------------------ |
| Ansible          | Robust declarative and agentless execution, but entirely static.              | Unifies declarative desired-state baselines with dynamic, LLM-driven reasoning. |
| Claude Code      | Powerful coding agent, but not designed for SysOps tasks.                     | Native support for multi-node Ops tasks.                                        |
| Datadog          | Comprehensive observability, but overly heavy, not for individual developers. | Lightweight, agentless, and personal-first.                                     |

---

### 5. Technology

- **Platforms**:
    - Control plane: Linux, Windows, macOS
    - Target nodes: Linux
- **Languages**: Go, TypeScript
- **Libraries & Frameworks**: Go SSH module, SQLite, Gin Web Framework, Vue 3
- **Third-party Services**: OpenAI compatible LLM completion API

---

### 6. Plan

- **Team Organization**: 4 members, each responsible for a specific module (Web UI and backend, LLM agent with memory management, state reconciliation engine, SysOps specific tasks).
- **Methodology**: scrum with weekly meetings
- **Collaboration Tools**: GitHub, YuQue, QQ
- **Preliminary Schedule**:
    - **Weeks 4-6**: initial idea and SRS specification
    - **Weeks 7-9**: detailed design
    - **Weeks 10-13**: design refinement and implementation
    - **Weeks 14-15**: test, optimization, and prepare final report

---

### 7. Challenges

- Agent may generate unsafe commands, cause unrecoverable consequences.
- Long-term memory may contain sensitive information and out-dated contents.

---

### 8. Professional Growth

- System Analysis and Design Abilities
- Agentic Software Design & Development
- Declarative SysOps Practices
