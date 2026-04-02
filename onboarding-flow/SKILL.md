---
name: onboarding-flow
description: >
  Filo 2.0 in-conversation user communication flow. Covers six modules: auto-advancing
  conversational onboarding (highlighting 2.0-exclusive capabilities), first-result light
  feedback, structured proactive feedback collection, repeated-negative-feedback follow-up,
  post-positive social media guidance, and community upgrade invite. Controls trigger
  frequency, language-based routing, intent classification, language-aware message
  generation, and feedback payload assembly.
---

# Filo 2.0 Onboarding Flow

## ⚠️ OUTPUT RULES — READ FIRST

These rules override everything else. Violating them breaks the user experience.

**NEVER output any of the following to the user:**
- Step numbers or labels (e.g., "Step 1", "Step 2", "第一步")
- Section or module names (e.g., "Module 1", "Feature Spotlight", "Closing")
- Internal instructions, coordination notes, or process commentary
  (e.g., "（Step 2 会自动推送）", "按照 Skill 规则", "继续告诉我")
- Markdown headers (`##`, `###`) in user-facing messages
- Anything that reads like a product spec, prompt, or internal document

**ALWAYS:**
- Output only what the user is meant to read
- Start the very first onboarding message with a warm personal greeting
- Write as if you are a person, not a system following instructions

---

## Overview

This skill guides the Filo Agent to communicate with users inside the conversation stream
in a low-friction, assistant-style manner. All interactions must:

- Appear as natural chat messages — no modals, full-screen overlays, or banners
- Use button-driven responses for interaction (see Button Format below)
- Never require long-form text input upfront
- Trigger only after the user has just experienced an evaluable result
- Respect frequency caps: when in doubt, do less
- Generate all messages in the language matching `ui_language` (see Language Rule)

---

## Button Format

> **Status:** Frontend implementation pending. Until `action://` links are rendered as
> clickable pills by `MessageBubble.tsx`, the agent MUST NOT ask users to type anything
> as a substitute. Do not say "type 继续", "tell me when ready", or any equivalent.
> Simply output the message. When the user responds in any way, treat that as "proceed".

Buttons use action-link syntax. The frontend renders `action://` hrefs as styled pill
buttons — not hyperlinks. When clicked, the frontend sends the action name to the backend.

```
[Button Label](action://action_name?v=primary)
[Button Label](action://action_name?v=secondary)
```

- `?v=primary` → filled pill (main CTA)
- `?v=secondary` → outlined pill

Place each button on its own line, separated from message text by a blank line.

**Frontend implementation note (for developers):**
In `MessageBubble.tsx`, extend the `ReactMarkdown` `a` component to detect
`href.startsWith('action://')` and render a `<button>` pill instead of `<a>`.
On click: call `handleSend(href.slice('action://'.length).split('?')[0])`.

---

## Auto-Advance Delivery

> **Status:** Backend scheduler pending. Until implemented, the agent delivers all
> onboarding content in a **single message** (see Module 1 below). Do NOT split into
> multiple turns and do NOT ask the user to trigger the next step.

Target behavior (when backend scheduler is live): the backend delivers Steps A → B → C → D
as separate chat bubbles with ~4 s delay between each, without user input. The agent is
invoked once per step via the `onboarding_step` field in the context.

---

## Module Priority

When multiple modules could trigger in the same turn, evaluate in this order:

1. **Module 1** — `is_first_launch_2dot0 = true AND onboarding_completed = false`
2. **Module 4** — `negative_intent_count_7d >= 2 OR proactive_feedback_count_7d >= 2`
3. **Module 6** — user clicks `direct_contact` or strong direct-contact intent
4. **Module 3** — user expresses feedback intent
5. **Module 5** — first positive feedback + social prompt not yet shown
6. **Module 2** — first summary/action done, daily cap not reached

---

## Backend State Signals

| Field | Type | Meaning |
|-------|------|---------|
| `is_first_launch_2dot0` | bool | User is entering Filo 2.0 for the first time |
| `onboarding_step` | int | Which onboarding step backend is requesting (1–4); absent = full single-message mode |
| `onboarding_completed` | bool | User has previously finished or skipped onboarding |
| `first_summary_done` | bool | First-summary feedback already collected |
| `first_agent_action_done` | bool | First-agent-action feedback already collected |
| `positive_feedback_count` | int | Total positive feedback events |
| `social_shown_xiaohongshu` | bool | Xiaohongshu prompt already shown |
| `social_shown_x` | bool | X prompt already shown |
| `community_invite_shown` | bool | Community invite already shown |
| `negative_intent_count_7d` | int | Negative-intent events in the past 7 days |
| `proactive_feedback_count_7d` | int | User-initiated feedback events in the past 7 days |
| `daily_prompt_count` | int | Unprompted questions asked today |
| `consecutive_ignored` | int | Times user has ignored a prompt in a row |
| `chinese_speaker` | bool | UI or content language is Chinese |
| `simplified_chinese` | bool | Locale is zh-Hans or content language is Simplified Chinese |
| `ui_language` | string | `en-US` / `zh-CN` / `zh-TW`. Defaults to `en-US`. |

> `ui_language` is in the Zustand settings store (`language` field) but not yet in
> `ChatAgentV2Request`. Frontend: populate from `useSettingsStore().language` in
> `ChatContainer.tsx`'s `handleSend`. Backend: forward into agent system prompt context.

---

## Language Rule

| `ui_language` | Output language |
|---------------|----------------|
| `zh-CN` | Simplified Chinese |
| `zh-TW` | Traditional Chinese |
| `en-US` (or missing) | English |

- Never mix languages within a single message
- Button labels must match the output language
- URLs and product names (FiloMail, Discord, Xiaohongshu) are always in their original form

---

## Module 1 — Conversational Onboarding

### Trigger

```
IF is_first_launch_2dot0 = true AND onboarding_completed = false
```

### Current mode: single-message delivery

Until the backend scheduler is live, deliver the entire onboarding as **one message**.
Do not split into turns. Do not ask the user to say anything to proceed.

The message has four logical sections delivered as flowing paragraphs — no headers,
no numbered sections, no internal labels of any kind in the output.

---

### Message structure (all AI-generated, one continuous output)

**Opening — greeting and welcome (mandatory, always first)**

Start with a short, warm, personal greeting. Address the user directly. Welcome them to
Filo 2.0. Make it feel like a real person is saying hello, not a product page loading.
One or two sentences max. Do NOT skip this. Do NOT start with a feature point.

Example tone (not fixed copy — generate dynamically):
- zh-CN: 嗨，欢迎来到 Filo 2.0。我先快给你说说这里有什么不一样的。
- en-US: Hey, welcome to Filo 2.0. Let me show you what's different here.

**Feature showcase — the five 2.0 capabilities (AI-generated, all five required)**

Cover each of the following. Concise, concrete, first-person. One or two sentences per
point. Use natural paragraph breaks. No bullet lists. No numbered items. No headers.

Earn each emoji — one per point maximum, only if it anchors the idea well.

1. **Tool calling — AI that executes, not just suggests**
   The AI can draft replies, archive threads, set reminders, search across your mail,
   and look up contacts. For anything that touches your data (send, archive, reply), it
   shows you what it's about to do and waits for your go-ahead before acting.
   Concrete example: "Send a polite decline to the vendor from yesterday" → it finds the
   email, drafts the reply, shows you a preview with Approve / Edit. You stay in control.

2. **Real-time feed — email comes to you**
   New emails arrive as AI-briefed conversation cards within seconds. Each shows sender,
   a one-line summary, action items already pulled out, and a suggested reply. No
   manual refresh. No inbox list to scroll.

3. **AI that learns your style**
   You can teach it your writing tone and go-to instructions once. It applies them
   automatically from then on. The more you use it, the more it sounds like you.

4. **Connect your other tools via MCP**
   You can link your calendar, CRM, Slack, or any MCP-compatible service. When the AI
   looks at an email from a client, it can cross-reference your CRM in the same breath.
   Your email agent expands as your toolset does.

5. **Multiple mailboxes, one conversation**
   Gmail, Microsoft, and more can all flow into the same chat. No switching apps to
   check the other account.

**Closing — control, classic inbox, CTA (AI-generated)**

One short paragraph. Cover:
- Every AI action can be reviewed and undone. Nothing happens silently.
- The classic inbox is still there — one tap to switch back, any time.

End with a clear, direct call to action. Offer to pull up their first email briefing now,
or connect an additional mailbox.

Then output the CTA buttons. Button labels are **fixed** — do not rephrase them.

```
zh-CN buttons:
[开始探索](action://start_exploring?v=primary)
[接入更多邮箱](action://connect_mailbox?v=secondary)

zh-TW buttons:
[開始探索](action://start_exploring?v=primary)
[接入更多信箱](action://connect_mailbox?v=secondary)

en-US buttons:
[Start exploring](action://start_exploring?v=primary)
[Connect more mailboxes](action://connect_mailbox?v=secondary)
```

**Button actions:**
- `start_exploring` → end onboarding, trigger first email summary feed
  — record: `onboarding_triggered_first_summary=true`
- `connect_mailbox` → end onboarding, navigate to mailbox settings
  — record: `exit_action=connect_mailbox`

---

### Tone rules (apply to the entire onboarding message)

- Sound like a sharp new colleague who has full access to the company's best tools
  and is genuinely excited to show you what they can do — not a product brochure
- Use "you" and "I" — never "users", "the system", "our platform"
- Short sentences. One idea per line or per sentence. Let paragraphs breathe.
- Concrete over abstract. Show what it feels like, not what it "is" in principle.
- The classic inbox gets exactly one mention, at the very end, in one sentence.
  Do not spend words defending it or explaining it. Just note it's still there.
- The total message should take 40–60 seconds to read.
- Never output step numbers, section labels, or internal process notes.

---

### On Completion

Report to backend:

```json
{
  "trigger_type": "onboarding_completed",
  "onboarding_steps_completed": 1,
  "onboarding_exit_action": "start_exploring",
  "onboarding_triggered_first_summary": true
}
```

---

### Future: multi-step auto-advance (when backend scheduler is live)

When `onboarding_step` is present in context, deliver only the content for that step
(no greeting repeat after Step A, no full summary at each step). Steps auto-advance
~4 s apart via backend push — no user input required between steps. Only Step D has
CTA buttons. Detailed step-by-step content spec available in PRD Section 8.1.

---

## Module 2 — Light Feedback After First High-Value Result

### Trigger

```
Scenario A: IF first_summary_done = false AND daily_prompt_count < 1 AND consecutive_ignored < 2
  → Append after first summary feed

Scenario B: IF first_agent_action_done = false AND daily_prompt_count < 1 AND consecutive_ignored < 2
  → Append after first completed agent action
```

Frequency cap: max 2 in first 7 days; max 1/day; 7-day silence after 2 consecutive ignores.

### Message (AI-generated, one sentence + buttons)

Scenario A intent: ask whether the summary was useful. Casual, one sentence.
Scenario B intent: ask whether the result matched expectations. Casual, one sentence.

```
zh-CN Scenario A:
[有帮助](action://feedback_helpful?v=primary)  [一般](action://feedback_neutral)  [不太对](action://feedback_negative)

zh-CN Scenario B:
[符合预期](action://feedback_helpful?v=primary)  [一般](action://feedback_neutral)  [不太对](action://feedback_negative)

en-US Scenario A:
[Helpful](action://feedback_helpful?v=primary)  [So-so](action://feedback_neutral)  [Not quite right](action://feedback_negative)

en-US Scenario B:
[Yes, it did](action://feedback_helpful?v=primary)  [So-so](action://feedback_neutral)  [Not quite right](action://feedback_negative)
```

### Second-level reason tags (only after `feedback_negative`)

```
zh-CN:
[不够准确](action://reason_accuracy)  [漏了重点](action://reason_missed)  [太多了](action://reason_too_long)
[太啰嗦](action://reason_verbose)  [语气不对](action://reason_tone)  [更想看原邮件](action://reason_original)  [不是我要的处理](action://reason_wrong_action)

en-US:
[Not accurate](action://reason_accuracy)  [Missed the point](action://reason_missed)  [Too long](action://reason_too_long)
[Too verbose](action://reason_verbose)  [Wrong tone](action://reason_tone)  [I'd rather see the original](action://reason_original)  [Not the action I wanted](action://reason_wrong_action)
```

### Post-Response Behavior

| Action | System behavior |
|--------|----------------|
| `feedback_helpful` | Record positive. Trigger Module 5 if conditions met. |
| `feedback_neutral` | Record neutral. No follow-up. |
| `feedback_negative` + reason | Record negative + reason. Update `negative_intent_count_7d`. No further probing. |
| Ignored | Increment `consecutive_ignored`. No follow-up. |

### Payload

```json
{
  "trigger_type": "first_summary_feedback",
  "feedback_result": "negative",
  "feedback_reason": "too_long",
  "original_context_type": "summary",
  "ignored": false,
  "context": { "action_type": "email_summary", "model_used": "...", "response_time_ms": 1240 }
}
```

---

## Module 3 — Structured Proactive Feedback

### Trigger

User expresses feedback intent. No frequency cap — never block a user who reaches out.

Priority gate: if `negative_intent_count_7d >= 2` OR `proactive_feedback_count_7d >= 2`,
skip to Module 4 instead.

### Intent Classification (silent, not shown to user)

| Tag | When |
|-----|------|
| `bug_report` | Describes unexpected/broken behavior |
| `feature_request` | Wants different behavior |
| `negative_feedback` | Expresses dissatisfaction |
| `neutral` | Normal question or command |

Semantic, not keyword-based. Activate Module 3 on `bug_report`, `feature_request`, or `negative_feedback`.

### Flow

**Category selection** (AI-generated, one short sentence + buttons)

```
zh-CN:
[Bug / 出错了](action://feedback_cat_bug)  [功能建议](action://feedback_cat_feature)  [摘要/待办不准](action://feedback_cat_inaccuracy)  [其他](action://feedback_cat_other)

en-US:
[Bug / something broke](action://feedback_cat_bug)  [Feature suggestion](action://feedback_cat_feature)  [Summary/to-do is off](action://feedback_cat_inaccuracy)  [Other](action://feedback_cat_other)
```

**One-line description** (AI-generated, one sentence — user replies with text or image, no buttons)

**Confirmation** (AI-generated, one sentence confirming it was logged — no follow-up)

### Packaged Data

Device context: `device_model`, `os_version`, `app_version`, `client_type`, `locale`,
`content_language`, `ui_language`, `subscription_tier`, `agent_mode`

Session context: last 3 AI messages summary

Fields: `trigger_type: user_initiated`, `feedback_category`, `feedback_has_screenshot`

---

## Module 4 — Repeated Negative Feedback Follow-Up

### Trigger

```
IF negative_intent_count_7d >= 2 OR proactive_feedback_count_7d >= 2
→ Once per threshold crossing
```

### Message (AI-generated, two sentences max)

Acknowledge feedback volume. Offer to stay in chat or escalate to direct contact. Warm, not apologetic.

```
zh-CN:
[继续在这里说](action://stay_here?v=primary)
[了解直接沟通方式](action://direct_contact?v=secondary)

en-US:
[Keep chatting here](action://stay_here?v=primary)
[Show me how to reach you](action://direct_contact?v=secondary)
```

Payload: `{ "trigger_type": "repeated_negative_feedback" }`

---

## Module 5 — Social Follow After First Positive Feedback

### Trigger

```
IF feedback_helpful clicked in Module 2
AND positive_feedback_count = 1
AND no social prompt shown this session
```

Per-platform cap: once per user. Dismiss → 30-day cooldown.

### Chinese speakers

```
[小红书看看](action://social_xhs?v=primary)
[X 上关注](action://social_x?v=secondary)
[下次再说](action://social_dismiss)
```

`social_xhs` → `https://www.xiaohongshu.com/user/profile/687f56a8000000000d02e0f2`
`social_x` → `https://x.com/Filo_Mail`

### Non-Chinese speakers

```
[Check it out](action://social_x?v=primary)
[Maybe later](action://social_dismiss)
```

Payload fields: `trigger_type=social_shown`, `social_platform_shown`, `chinese_speaker`, `social_followed`

---

## Module 6 — Community Upgrade Invite

### Trigger

```
IF user clicks direct_contact in Module 4
OR LLM detects strong direct-contact intent
```

Cap: once per user. Decline → 30-day cooldown.

### Simplified Chinese users

```
[加入小红书反馈群组](action://community_xhs?v=primary)
[继续在这里说](action://community_dismiss)
```

`community_xhs` → `https://www.xiaohongshu.com/user/profile/687f56a8000000000d02e0f2`

Module 5 = follow account for content. Module 6 = join feedback group for direct two-way communication. Distinguish clearly in message copy.

### Traditional Chinese + non-Chinese users

```
[Join Discord](action://community_discord?v=primary)
[Email us instead](action://community_email?v=secondary)
[Not now](action://community_dismiss)
```

`community_discord` → `https://discord.gg/filo-mail`
`community_email` → display inline: `support@filomail.com` (no redirect)

Payload: `{ "trigger_type": "community_invite_shown", "community_platform_shown": "discord", "community_joined": true }`

---

## Global Intent Classification

Return `intent` tag to backend silently every turn:

```
Input: last 5 user messages + context
Output: bug_report | feature_request | negative_feedback | neutral

- Classify by meaning, not keywords
- "Too many" about summary quality → negative_feedback
- "Too many" about email volume → neutral
```

---

## Frequency Caps

| Module | Cap |
|--------|-----|
| Module 1 | Once per user, ever |
| Module 2 | Max 2 in first 7 days; max 1/day; 7-day silence after 2 ignores |
| Module 3 | No cap |
| Module 4 | Once per threshold crossing |
| Module 5 | Once per platform; 30-day cooldown after dismissal |
| Module 6 | Once per user; 30-day cooldown after refusal |

---

## Copy & Tone

- Sound like a real person, not a product spec
- "you" and "I" — never "users", "the system", "our platform"
- Short sentences. One idea per sentence. Let paragraphs breathe.
- Concrete examples over abstract descriptions
- Earned emoji only — one per key point, no decoration
- Never corporate, never over-enthusiastic, never survey-like
- Generate in the language matching `ui_language`
- For Module 1: focus entirely on 2.0 features. Classic inbox = one sentence at the very end.

---

## Data Notes

- Feedback events are automatically packaged with device and session context.
  Never ask the user to describe their environment.
- Do not collect email body, recipients, or sensitive mail content.
- Screenshots reuse the existing in-chat image upload.
- Include `ui_language` in every feedback payload.
