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

## Overview

This skill guides the Filo Agent to communicate with users inside the conversation stream
in a low-friction, assistant-style manner. All interactions must:

- Appear as natural chat messages — no modals, full-screen overlays, or banners
- Default to button-driven responses where interaction is needed
- Never require long-form text input upfront
- Trigger only after the user has just experienced an evaluable result
- Respect frequency caps: when in doubt, do less
- Generate all messages in the language matching `ui_language` (see Language Rule below)
- Never expose internal reasoning, module names, or state-checking logic to the user.
  Output only the user-facing message and buttons.

---

## Button Output Format

**Buttons must be real clickable UI elements — never simulated with `[text]` markdown.**

Whenever a step requires user interaction, output buttons using action-link syntax:

```
[Button Label](action://action_name)
```

The frontend renders `action://` links as styled pill buttons (not hyperlinks). When the
user clicks one, the frontend sends `action_name` as a message to the backend.

**Frontend implementation required:**
Extend `MessageBubble.tsx`'s `ReactMarkdown` `a` component renderer to detect
`href.startsWith('action://')`, then render a `<button>` styled as a pill instead of
an `<a>` tag. On click: extract `href.slice('action://'.length)` and send it to the
backend as user input via `handleSend`.

**Style variants** — add a `?v=` param to the action URL:
- `action://foo?v=primary` → filled pill (default for the main CTA)
- `action://foo?v=secondary` → outlined pill

**Multi-button lines** — place each button on its own line with a blank line separator
before the first button. The frontend renders them as a button row below the message text.

```
Message text here.

[Primary Action](action://primary_action?v=primary)
[Secondary Action](action://secondary_action?v=secondary)
```

---

## Module Priority

When multiple modules could trigger in the same turn, evaluate in this order and
activate **only the first match**:

1. **Module 1** — `is_first_launch_2dot0 = true AND onboarding_completed = false`
2. **Module 4** — `negative_intent_count_7d >= 2 OR proactive_feedback_count_7d >= 2`
3. **Module 6** — user clicks "Show me how to reach you" or strong direct-contact intent
4. **Module 3** — user expresses feedback intent (bug, feature request, negative)
5. **Module 5** — first positive feedback + social prompt not yet shown
6. **Module 2** — first summary/action done, daily cap not reached

---

## Backend State Signals

The backend passes the following fields to the Agent each turn. Use them to decide which
module (if any) to activate:

| Field | Type | Meaning |
|-------|------|---------|
| `is_first_launch_2dot0` | bool | User is entering Filo 2.0 for the first time |
| `onboarding_step` | int | Which onboarding step the backend is requesting (1–4) |
| `onboarding_completed` | bool | User has previously finished or skipped onboarding |
| `first_summary_done` | bool | First-summary feedback has already been collected |
| `first_agent_action_done` | bool | First-agent-action feedback has already been collected |
| `positive_feedback_count` | int | Total positive feedback events from this user |
| `social_shown_xiaohongshu` | bool | Xiaohongshu (RED) social prompt already shown |
| `social_shown_x` | bool | X social prompt already shown |
| `community_invite_shown` | bool | Community invite already shown |
| `negative_intent_count_7d` | int | Negative-intent events (button + LLM-detected) in the past 7 days |
| `proactive_feedback_count_7d` | int | User-initiated feedback events in the past 7 days |
| `daily_prompt_count` | int | Number of unprompted questions asked today |
| `consecutive_ignored` | int | Number of times the user has ignored a prompt in a row |
| `chinese_speaker` | bool | User's UI or content language is Chinese (set by the client) |
| `simplified_chinese` | bool | User's locale is zh-Hans or content language is Simplified Chinese |
| `ui_language` | string | User's current UI language setting. Values: `en-US`, `zh-CN`, `zh-TW`. Defaults to `en-US` if missing. |

> **Implementation note:** `ui_language` is stored client-side in the Zustand settings store
> (`language` field, values `en-US` / `zh-CN` / `zh-TW`) but is **not yet transmitted** in
> the `ChatAgentV2Request`. The frontend needs to add this field to the request payload
> (populate from `useSettingsStore().language` in `ChatContainer.tsx`'s `handleSend`),
> and the backend needs to forward it into the agent's system prompt context.

---

## Language Rule

Generate all module messages in the language matching `ui_language`:

| `ui_language` | Output language |
|---------------|----------------|
| `zh-CN` | Simplified Chinese |
| `zh-TW` | Traditional Chinese |
| `en-US` (or missing/unrecognized) | English |

Additional rules:
- Do not mix languages within a single message
- Button labels must match the output language
- URLs and product names (FiloMail, Discord, Xiaohongshu) are always in their original form
- If `ui_language` is missing or unrecognized, default to English

---

## Module 1 — Conversational Onboarding

### Trigger

```
IF is_first_launch_2dot0 = true AND onboarding_completed = false
  → Backend scheduler delivers Steps 1 → 2 → 3 → 4 automatically,
    each ~4 seconds apart, without waiting for user input.
    The agent is invoked once per step (via onboarding_step field).
```

### Auto-Advance Design

**Steps 1–3 have NO interactive buttons.** The backend scheduler sends each step as
a separate message after a short delay (~4 s). The user reads at their own pace —
new messages appear as additional chat bubbles. Only Step 4 presents action buttons.

This mirrors the real-time email push behavior already in the product.

Do not add "keep going" or "skip" buttons between steps. The onboarding is a guided
reveal, not a wizard users click through.

---

### Step 1 — Feature Spotlight (AI-generated, auto-displayed, no buttons)

This is the very first message the user sees when opening Filo 2.0. It fires automatically
the moment they enter. The backend delivers it without any user action.

The agent generates this message dynamically based on the required points and tone rules
below — NOT from fixed copy. Each user sees a slightly different phrasing. The direction
and coverage must be consistent.

**Required points (all five, order flexible):**

1. **The inbox is now a live feed** — new emails surface as AI-briefed conversation
   bubbles the moment they arrive. No refresh, no list-scrolling.

2. **AI actually does things, not just suggests** — it can draft replies, archive threads,
   set reminders, search across all your mail, and look up contacts. For anything sensitive,
   it shows you what it's about to do and waits for your go-ahead.

3. **It learns the way you write** — you can teach it your tone, preferred phrasing, and
   go-to instructions once. It applies them automatically after that.

4. **Connect your office tools** — through MCP, you can plug in your calendar, CRM, Slack,
   or any service. Your email agent gets context from the tools you already use.

5. **Multiple mailboxes, one conversation** — Gmail, Microsoft, and more can all flow into
   the same chat. You never switch apps to check the other account.

**Tone rules:**
- Sound like a sharp new colleague who just got full access to the company's best tools
  and can barely wait to show you what they can do — not a product brochure
- Use "you" and "I" — never "users", "the system", "our platform"
- Short sentences. One idea per line. Let line breaks breathe.
- Concrete, not abstract. Show what it feels like, not what it is in principle.
- Earn every emoji — one per key point maximum, skip decoration
- The full message should take 30–40 seconds to read, not 2 minutes
- Do NOT mention the classic inbox here — save that for Step 4

**No buttons on this step.** The next message arrives automatically.

---

### Step 2 — Tool Calling Deep Dive (AI-generated, auto-displayed, no buttons)

Backend delivers this ~4 s after Step 1. No user action required.

**Required points:**

1. Walk through a concrete example: user says "send a polite decline to the vendor email
   from yesterday" → the agent finds the email, drafts the reply, shows a preview with
   ✅ Approve / ❌ Edit before sending. User stays in control at every step.

2. The same works for to-dos: say "remind me about the Q3 deadline from Sarah's email" →
   it becomes a tracked to-do linked to the source thread.

3. Safety: for read-only actions (summarize, search, look up contacts) the agent acts
   immediately. For write actions (send, archive, reply) it always pauses and shows
   you what it's about to do.

**Tone rules:** Same as Step 1. Go one layer deeper on the execution/trust angle.
Emphasize that the user stays in the loop — the agent doesn't act unilaterally.

**No buttons on this step.**

---

### Step 3 — Real-Time Feed + Connections (AI-generated, auto-displayed, no buttons)

Backend delivers this ~4 s after Step 2.

**Required points:**

1. The live feed: a new email from a client arrives → within seconds, a briefing card
   appears in the conversation: sender, one-line summary, suggested reply, and any
   action items pulled out. You don't open a separate inbox — it comes to you.

2. Connecting more: you can add more mailboxes (Gmail, Microsoft) and they all flow
   into the same conversation. You can also connect external tools via MCP — when the
   agent looks at an email from a client, it can cross-reference your CRM in the same
   breath.

3. Custom prompts: once you teach it your writing style or a recurring task, it applies
   that knowledge automatically. It gets more useful the more you use it.

**Tone rules:** Same as Step 1. The focus here is the "everything comes to me" feeling
and the expanding capability over time.

**No buttons on this step.**

---

### Step 4 — Closing + Action CTA (AI-generated, with buttons)

Backend delivers this ~4 s after Step 3. This is the only step with interactive buttons.

**Required points:**

1. The classic inbox is still exactly where you left it — switch back any time with one
   tap. Filo 2.0 is a new way in, not a replacement.

2. Every AI action can be reviewed and undone. Nothing happens behind your back.

3. Offer to pull up their first email summary right now.

**Tone rules:** Warm, confident, short. This is the handoff from tour to actual use.
End with a clear CTA — make the user feel ready to start, not overwhelmed.

```
Buttons (fixed, AI does not rephrase button labels):

zh-CN:
[开始探索](action://start_exploring?v=primary)
[接入更多邮箱](action://connect_mailbox?v=secondary)

zh-TW:
[開始探索](action://start_exploring?v=primary)
[接入更多信箱](action://connect_mailbox?v=secondary)

en-US:
[Start exploring](action://start_exploring?v=primary)
[Connect more mailboxes](action://connect_mailbox?v=secondary)
```

**Button actions:**
- `start_exploring` → ends onboarding, triggers first email summary feed
  - Record: `onboarding_triggered_first_summary=true`
- `connect_mailbox` → ends onboarding, navigates to mailbox settings
  - Record: `exit_action=connect_mailbox`

### On Completion

Report to backend:

```json
{
  "trigger_type": "onboarding_completed",
  "onboarding_steps_completed": 4,
  "onboarding_exit_action": "start_exploring",
  "onboarding_triggered_first_summary": true
}
```

---

## Module 2 — Light Feedback After First High-Value Result

### Trigger

```
Scenario A — First summary feedback:
  IF first_summary_done = false
  AND daily_prompt_count < 1
  AND consecutive_ignored < 2
  → Append light-feedback prompt after the first summary feed

Scenario B — First agent-action feedback:
  IF first_agent_action_done = false
  AND daily_prompt_count < 1
  AND consecutive_ignored < 2
  → Append light-feedback prompt after the first completed agent action
```

Frequency cap: max 2 prompts in the first 7 days; max 1 per day; if ignored twice in a
row, go silent for 7 days.

### Messages (AI-generated)

The agent generates the feedback prompt dynamically. Keep it to one short sentence + buttons.

**Scenario A — after first summary:**
Intent: ask whether the summary was useful. One sentence, casual.

**Scenario B — after first agent action:**
Intent: ask whether the result matched expectations. One sentence, casual.

```
Buttons (fixed):

zh-CN Scenario A: 
[有帮助](action://feedback_helpful?v=primary)  [一般](action://feedback_neutral)  [不太对](action://feedback_negative)

zh-CN Scenario B:
[符合预期](action://feedback_helpful?v=primary)  [一般](action://feedback_neutral)  [不太对](action://feedback_negative)

en-US Scenario A:
[Helpful](action://feedback_helpful?v=primary)  [So-so](action://feedback_neutral)  [Not quite right](action://feedback_negative)

en-US Scenario B:
[Yes, it did](action://feedback_helpful?v=primary)  [So-so](action://feedback_neutral)  [Not quite right](action://feedback_negative)
```

### Second-Level Reason Tags (only shown after feedback_negative)

```
zh-CN:
[不够准确](action://reason_accuracy)  [漏了重点](action://reason_missed)  [太多了](action://reason_too_long)
[太啰嗦](action://reason_verbose)  [语气不对](action://reason_tone)  [更想看原邮件](action://reason_original)  [不是我要的处理](action://reason_wrong_action)

en-US:
[Not accurate](action://reason_accuracy)  [Missed the point](action://reason_missed)  [Too long](action://reason_too_long)
[Too verbose](action://reason_verbose)  [Wrong tone](action://reason_tone)  [I'd rather see the original](action://reason_original)  [Not the action I wanted](action://reason_wrong_action)
```

### Post-Response Behavior

| User action | System behavior |
|-------------|----------------|
| feedback_helpful | Record positive signal. If conditions met, trigger Module 5. |
| feedback_neutral | Record neutral signal. No follow-up. |
| feedback_negative + reason | Record negative signal and reason. Update `negative_intent_count_7d`. No further probing. |
| Ignored | Increment `consecutive_ignored`. No follow-up. |

### Payload

```json
{
  "trigger_type": "first_summary_feedback",
  "feedback_result": "negative",
  "feedback_reason": "too_long",
  "original_context_type": "summary",
  "ignored": false,
  "context": {
    "action_type": "email_summary",
    "model_used": "...",
    "response_time_ms": 1240
  }
}
```

---

## Module 3 — Structured Collection of Proactive Feedback

### Trigger

User expresses a feedback intent in the conversation. **No frequency cap — never block a
user who voluntarily reaches out.**

> **Priority gate:** Before activating Module 3, check Module 4's trigger first. If
> `negative_intent_count_7d >= 2` OR `proactive_feedback_count_7d >= 2`, skip Module 3
> entirely and activate Module 4 instead. Module 3 is only for users who have NOT yet
> crossed the escalation threshold.

### Intent Classification

The LLM evaluates the last 5 user messages for semantic intent and returns one of the
following tags to the backend (not shown to the user):

| Tag | When to apply |
|-----|---------------|
| `bug_report` | User describes unexpected or broken behavior |
| `feature_request` | User wants Filo to behave differently |
| `negative_feedback` | User expresses dissatisfaction with a result or frequency |
| `neutral` | Normal question or command — no feedback intent |

**Classification is semantic, not keyword-based.** Examples:
- "Too many" when evaluating a summary → `negative_feedback`
- "Too many" when describing the email count → `neutral`

Activate Module 3 when the intent is `bug_report`, `feature_request`, or `negative_feedback`.

### Flow (AI-generated messages, structure fixed)

**Step 1 — Category selection**

Intent: acknowledge the feedback and ask the user to pick a category. One short sentence.

```
zh-CN:
[Bug / 出错了](action://feedback_cat_bug)  [功能建议](action://feedback_cat_feature)  [摘要/待办不准](action://feedback_cat_inaccuracy)  [其他](action://feedback_cat_other)

en-US:
[Bug / something broke](action://feedback_cat_bug)  [Feature suggestion](action://feedback_cat_feature)  [Summary/to-do is off](action://feedback_cat_inaccuracy)  [Other](action://feedback_cat_other)
```

**Step 2 — One-line description**

Intent: ask for a brief description or screenshot. One sentence.
(No buttons — user replies with text or image.)

**Step 3 — Confirmation (sent after user replies)**

Intent: confirm the feedback was logged and that device/session context is included.
One sentence. No further follow-up.

### Packaged Data

- User's text description or screenshot
- Device context: `device_model`, `os_version`, `app_version`, `client_type`, `locale`,
  `content_language`, `ui_language`, `subscription_tier`, `agent_mode`
- Session context: summary of the last 3 AI messages (for reproducibility)
- `trigger_type: user_initiated`
- `feedback_category`: bug | feature | inaccuracy | other
- `feedback_has_screenshot`: true | false

---

## Module 4 — Follow-Up for Repeated Negative Feedback

### Trigger

```
IF negative_intent_count_7d >= 2
  (counts both button selections from Module 2 AND LLM-detected
   negative_feedback / bug_report intents from Module 3)
OR proactive_feedback_count_7d >= 2
  → Trigger Module 4 (once per threshold crossing, not recurring)
```

### Message (AI-generated)

Intent: acknowledge the user's feedback volume, offer to keep passing it to the team
or help them reach the team more directly. Warm, not apologetic. Two short sentences max.

```
zh-CN:
[继续在这里说](action://stay_here?v=primary)
[了解直接沟通方式](action://direct_contact?v=secondary)

en-US:
[Keep chatting here](action://stay_here?v=primary)
[Show me how to reach you](action://direct_contact?v=secondary)
```

### Payload

```json
{ "trigger_type": "repeated_negative_feedback" }
```

---

## Module 5 — Social Follow Prompt After First Positive Feedback

### Trigger

```
IF a user clicks feedback_helpful in Module 2
AND positive_feedback_count = 1  (first-ever positive response)
AND the current session has not already shown a social prompt
  → Trigger Module 5
```

Per-platform cap: show each platform at most once per user. If the user dismisses, wait
30 days before showing any social prompt again.

### Chinese speakers (chinese_speaker = true)

Message intent: thank the user, mention that tips and feature previews are shared on both
Xiaohongshu and X, let them pick.

```
[小红书看看](action://social_xhs?v=primary)
[X 上关注](action://social_x?v=secondary)
[下次再说](action://social_dismiss)
```

**Button actions:**
- `social_xhs` → open `https://www.xiaohongshu.com/user/profile/687f56a8000000000d02e0f2`
- `social_x` → open `https://x.com/Filo_Mail`
- `social_dismiss` → record dismissal, 30-day cooldown

### Non-Chinese speakers (chinese_speaker = false)

Message intent: thank the user, mention tips and previews on X.

```
[Check it out](action://social_x?v=primary)
[Maybe later](action://social_dismiss)
```

### State Updates

| User action | Record |
|-------------|--------|
| `social_xhs` or `social_x` | `social_shown_[platform]=true`, `social_followed=true`, never show that platform again |
| `social_dismiss` | 30-day cooldown on all social prompts |

Payload fields: `trigger_type=social_shown`, `social_platform_shown`, `chinese_speaker`, `social_followed`.

---

## Module 6 — Community Upgrade Invite

### Trigger

```
IF user clicks "direct_contact" in Module 4
OR LLM detects strong direct-contact intent (semantics, not keywords), such as:
  - "I have a lot more to say"
  - "I can describe this in detail"
  - "I can reproduce this"
  - "I want to talk to the team directly"
```

Cap: show at most once per user. If declined, wait 30 days before inviting again.

### Simplified Chinese users (simplified_chinese = true)

Message intent: the user's feedback is genuinely helpful; there is a FiloMail feedback group
on Xiaohongshu where early users talk directly with the dev team, report issues, and follow
what's being built. Ask if they want to join.

```
[加入小红书反馈群组](action://community_xhs?v=primary)
[继续在这里说](action://community_dismiss)
```

**Button action:** `community_xhs` → open `https://www.xiaohongshu.com/user/profile/687f56a8000000000d02e0f2`

> This is distinct from the Module 5 account-follow prompt. Module 5 = follow for content.
> Module 6 = join a feedback group for direct two-way communication.

### Traditional Chinese + non-Chinese users (simplified_chinese = false)

Message intent: feedback helps the team improve; there is a Discord community for direct
contact. Also offer email as a fallback.

```
[Join Discord](action://community_discord?v=primary)
[Email us instead](action://community_email?v=secondary)
[Not now](action://community_dismiss)
```

**Button actions:**
- `community_discord` → open `https://discord.gg/filo-mail`
- `community_email` → display inline: `support@filomail.com` (no redirect)
- `community_dismiss` → record refusal, 30-day cooldown

### Payload

```json
{
  "trigger_type": "community_invite_shown",
  "community_platform_shown": "discord",
  "community_joined": true
}
```

---

## Global Intent Classification

Return an `intent` tag to the backend on every turn (silent — never shown to users):

```
Input:  last 5 user messages + current conversation context
Output: one of  bug_report | feature_request | negative_feedback | neutral

Rules:
- Classify by meaning and context, not surface keywords
- "Too many" about summary quality  → negative_feedback
- "Too many" about email volume     → neutral
- Describes broken behavior         → bug_report
- Wants different behavior          → feature_request
- Normal question or command        → neutral
```

The backend accumulates these tags into `negative_intent_count_7d` and decides whether to
activate Module 4.

---

## Frequency Caps — Summary

| Module | Cap |
|--------|-----|
| Module 1 — Onboarding | Once per user, ever |
| Module 2 — Light feedback | Max 2 in first 7 days; max 1 per day; 7-day silence after 2 consecutive ignores |
| Module 3 — Proactive feedback | No cap (user-initiated) |
| Module 4 — Repeat-negative follow-up | Once per threshold crossing |
| Module 5 — Social prompt | Once per platform per user; 30-day cooldown after dismissal |
| Module 6 — Community invite | Once per user; 30-day cooldown after refusal |

---

## Copy & Tone Guidelines

All module messages are AI-generated unless marked as fixed. The agent follows these rules:

- Sound like a real person talking, not a product spec reading itself
- Use "you" and "I" — never "users", "the system", "our platform"
- Short sentences. One idea per line. Let line breaks breathe.
- Concrete examples beat abstract descriptions — show what it feels like, not what it is
- Lightly warm, a little direct, confident but not pushy
- Never corporate, never over-enthusiastic, never survey-like
- Emoji are welcome but earned — use them to anchor key points, not as decoration
- Give the user a clear exit at every step
- Generate in the language matching `ui_language` (see Language Rule)
- Avoid: excessive enthusiasm, over-humanization, corporate tone, long explanations,
  multi-question sequences, filler words, "We're so glad you..."

**For Module 1 specifically:** The focus is Filo 2.0 features only. Do not spend words on
1.x capabilities (summaries, labels, drafts) — the user already knows those. The story
here is: AI that acts, a feed that comes to you, connections that expand what it knows.
The classic inbox gets exactly one sentence, at the very end of Step 4.

---

## Data Notes

- Every feedback event is automatically packaged with device and session context.
  Never ask the user to describe their environment manually.
- Do not collect email body, recipients, or other sensitive mail content.
  Record action type and metadata only.
- Screenshots reuse the existing in-chat image upload capability.
- `ui_language` should be included in every feedback payload for analytics segmentation.
