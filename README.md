# filo-skills

**Claude Code skill library for [FiloMail](https://filomail.com).**

A collection of agent skills that power Filo's in-conversation user experience — onboarding, feedback collection, social growth, and community escalation — all delivered as natural chat messages without modals or banners.

---

**FiloMail 的 [Claude Code](https://claude.ai/code) Skill 库。**

一组 Agent Skill，驱动 Filo 在对话中完成用户引导、反馈收集、社交增长和社区升级——全部以自然聊天消息呈现，无弹窗、无横幅。

---

## Skills / Skill 列表

| Skill | Description |
|-------|-------------|
| [`onboarding-flow`](./onboarding-flow/SKILL.md) | Full in-conversation onboarding and feedback loop for Filo 2.0 |

---

## onboarding-flow

### English

A single skill that manages the entire post-launch communication lifecycle inside the Filo chat interface. It covers six modules:

| # | Module | Trigger |
|---|--------|---------|
| 1 | **Conversational Onboarding** | User opens Filo 2.0 for the first time |
| 2 | **Light Feedback After First Result** | After the first AI summary or agent action |
| 3 | **Structured Proactive Feedback** | User voluntarily expresses a bug, feature request, or complaint |
| 4 | **Repeated-Negative Follow-Up** | User has given negative signals 2+ times in 7 days |
| 5 | **Social Follow Prompt** | User gives their first positive feedback |
| 6 | **Community Upgrade Invite** | User signals they want direct contact with the team |

**Design principles:**
- All messages appear as natural chat — no modals, overlays, or push notifications
- Every step offers a clear exit; users are never trapped in a flow
- Messages are AI-generated and adapt to `ui_language` (`en-US`, `zh-CN`, `zh-TW`)
- Strict frequency caps prevent over-prompting
- No email body content or PII is ever collected; only action metadata

**Backend signals:** The agent reads state fields from each turn's context (`is_first_launch_2dot0`, `negative_intent_count_7d`, `ui_language`, etc.) and emits a structured JSON payload after each interaction for backend recording.

---

### 中文

一个统一管理 Filo 对话内全流程用户沟通的 Skill，覆盖六个模块：

| # | 模块 | 触发条件 |
|---|------|---------|
| 1 | **对话式引导（Onboarding）** | 用户首次进入 Filo 2.0 |
| 2 | **首次结果轻量反馈** | AI 完成第一次摘要或执行第一次操作后 |
| 3 | **主动反馈结构化收集** | 用户主动表达 Bug、功能建议或负面情绪 |
| 4 | **连续负面反馈跟进** | 7 天内收到 2 次及以上负面信号 |
| 5 | **社交关注引导** | 用户第一次给出正面反馈 |
| 6 | **社区升级邀请** | 用户明确表达希望与团队直接沟通 |

**设计原则：**
- 所有消息以普通聊天气泡呈现，不使用弹窗、全屏覆盖或横幅
- 每个步骤都提供明确的退出选项，用户不会被困在流程中
- 消息由 AI 动态生成，根据 `ui_language` 自动适配语言（`en-US` / `zh-CN` / `zh-TW`）
- 严格的频率限制防止过度打扰
- 不采集邮件正文内容或任何 PII，仅记录操作元数据

**后端信号：** Agent 从每轮 context 中读取状态字段（`is_first_launch_2dot0`、`negative_intent_count_7d`、`ui_language` 等），并在每次交互后输出结构化 JSON payload 供后端记录。

---

## Installation / 安装

These skills are loaded into the Filo agent's system context via [Claude Code's skill system](https://claude.ai/code).

这些 Skill 通过 [Claude Code 的 Skill 系统](https://claude.ai/code)加载到 Filo Agent 的系统上下文中。

```bash
# Install via Claude Code CLI
claude skill add https://github.com/justinbao19/filo-skills/tree/main/onboarding-flow
```

---

## License / 许可

Private — internal use only. / 私有仓库，仅限内部使用。
