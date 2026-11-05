# LLM Integration Plan

How Paathera can use a large language model (LLM) safely, which options to check, and the order to check them in. Written so you can work through it in your free time, one step at a time.

> Status: **planning only.** Nothing here is built yet. AI triage belongs to Phase 2 (see `CLAUDE.md`). Facts about prices, programs and regions were checked in November 2026. They change often, so re-check each one when you get to it.

---

## 1. What the LLM is allowed to do (and never do)

These come straight from the non-negotiable rules in `CLAUDE.md`. Every option below has to fit them.

| The LLM **may** | The LLM **may never** |
|---|---|
| Read a new user message and score how worrying it is (safety triage) | Replace the keyword crisis check. Keywords always run first, AI is an extra layer (rule 3) |
| Raise the alert level above what the keywords found | Lower an alert the keywords raised. AI can only escalate, never calm things down |
| Draft a reply that **you** read, edit and send | Send anything to a user by itself (rule 4) |
| Summarise a long conversation for you before you reply | Speak as you, or copy your voice. AI text shown to users is labelled "automated" (rule 5) |
| Suggest a quick reply or a helpline to mention | Process under-18 users' data (no accounts until Phase 4, rule 6) |

If an option can't meet the right-hand column, drop it.

---

## 2. Use cases, in the order to build them

| # | Use case | Who sees the output | Value | Risk | When |
|---|---|---|---|---|---|
| A | **Safety triage**: score every user message for risk, add an alert when it's worrying | Admin only (alerts) | High. Catches messages keywords miss: "I just want everything to stop", Hinglish, spelling slips | Medium. A miss is bad, but keywords still run | Phase 2, first |
| B | **Conversation summary**: "Here's what Priya has shared this week" before you reply | Admin only | High once you have many users | Low | Phase 2, second |
| C | **Draft replies**: suggested wording you edit before sending | Admin only, until you send it | Medium. Saves typing, but your own voice matters most | Medium. Drafts can sound generic or wrong. You always edit | Phase 2, third |
| D | **Language help**: a Hindi or Marathi message translated for you, or your reply translated back | Admin; user sees only what you send | Medium | Medium. Translation errors on feelings | Later, optional |

Speech-to-text and text-to-speech (Phase 1) are separate services, not LLMs. They stay behind their own interfaces and aren't covered here.

---

## 3. How it fits the architecture

### 3.1 Triage flow (use case A)

```
User sends message
      │
      ▼
Firestore: conversations/{uid}/messages/{id} created
      │
      ▼
Cloud Function onMessageCreated (asia-south1)
      │
      ├─► 1. Keyword check (always, synchronous, no network)  ──► crisis? ──► alert + crisis flag  (as today)
      │
      └─► 2. AI triage (async, with a short timeout)
              │
              ├─ success ─► final level = MAX(keyword level, AI level)
              │              if AI level is higher ─► new or upgraded alert, reason "ai"
              │
              └─ error / timeout / refusal ─► keep the keyword result, log "triage_unavailable"
                                              (no message text in logs), never block anything
```

Important points:

- **The keyword path never waits for the AI.** The user's crisis screen and your alert from keywords happen exactly as they do without AI.
- **Fail safe:** if the AI provider is down, slow or refuses, nothing changes for the user. Optionally, if triage fails three times in a row, send you a low-priority "AI triage is down" notice.
- **The AI only adds.** It can turn "no alert" into "alert" or "watch" into "urgent". It can never remove or downgrade an alert.
- **The AI never decides what the user sees.** The crisis screen on the user's phone is triggered only by the keyword check (on device and in the function) or by the user tapping "I need help now". An AI "high" result alerts **you**; you then decide whether to reach out. This keeps the user experience predictable and works offline.

  *Open question for you:* should an AI "imminent" result also show the user the crisis screen? Rule 2 says every crisis trigger does both. If AI counts as a trigger, the answer is yes, and the screen would appear a few seconds after sending. Decide this before building, and write it into `docs/SAFETY.md`.

### 3.2 Where results are stored

The user must never see AI scores about themselves, and today's rules say users can't set safety flags. So triage results go in a **separate admin-only collection**, not on the message:

```
triage/{messageId}
  uid, conversationId
  keywordLevel:  none | watch | urgent
  aiLevel:       none | low | medium | high | imminent | unavailable
  aiCategories:  ["self_harm", "hopelessness", "abuse", "postpartum", ...]
  aiReason:      short English reason for you (no quotes copied from the message)
  provider, model, promptVersion, latencyMs
  createdAt
```

Firestore rule: `allow read: if isAdmin(); allow write: if false;` (server-only writes). Add a rules test for it, as rule 9 requires.

Drafts (use case C) go in `drafts/{id}`, admin-only. When you press Send, the app writes a normal `sender: 'admin'` message containing **your edited text**. Nothing from `drafts` ever reaches the user's screen directly.

### 3.3 Keep the provider swappable

Write one small interface in `functions/src/safety/ai/` and one file per provider. Then switching from one provider to another is a config change, not a rewrite:

```ts
// functions/src/safety/ai/types.ts  (sketch, not final code)
export type AiLevel = 'none' | 'low' | 'medium' | 'high' | 'imminent';

export interface TriageResult {
  level: AiLevel;
  categories: string[];
  reason: string;          // for the admin, never shown to the user
  provider: string;
  model: string;
}

export interface TriageProvider {
  // Must resolve within the timeout or throw. Never returns message text.
  triage(input: { text: string; group: 'adult' | 'mother' | 'senior'; language?: string }): Promise<TriageResult>;
}
```

The provider name and model come from environment config. Any API key lives in Secret Manager (rule 8). If you choose Vertex AI, there's no key at all: the Cloud Function's service account is the login.

### 3.4 What gets sent to the provider (data minimisation)

Send only what's needed:

- **Triage:** the single new message, plus at most the previous 3–5 user messages for context. Before sending, strip phone numbers, emails and names with a simple regex pass. No user ID, no profile, no age.
- **Summary or draft:** the recent conversation with the same stripping. Use "User" and "Supporter" instead of names.
- **Never** send voice files, photos or journal entries. Journal entries are private unless shared, and even shared ones aren't needed.
- **Never** log the prompt or the response text. Log only level, latency, model and error codes.

---

## 4. Provider options to validate

Check each one against the same checklist (section 6). Don't commit to one until the evaluation in section 7 is done.

### Option 1: Claude via the Anthropic API (direct)

- **What:** Call Claude models directly from Cloud Functions with the official `@anthropic-ai/sdk` package.
- **Models and price per million tokens (input / output, November 2026):**
  - Claude Haiku 5.5: $0.10 / $0.50. Cheapest and fastest, the natural first candidate for triage.
  - Claude Sonnet 5.5: $2 / $10. Better judgement, a good candidate for summaries and drafts.
  - Claude Opus 5.5: $4 / $20. Most capable of the three. Worth testing whether the extra quality matters for your hardest cases.
- **Pros:** Strong on careful, sensitive conversations. Structured outputs give you guaranteed JSON for triage. Newest models arrive here first.
- **Check:** where inference runs (the API offers `global` or US-only, not India), data-retention terms, whether zero data retention is available to you, and how the models handle Hindi, Marathi and Hinglish.
- **Billing:** card on file, prepaid credits. A separate bill from Google.

### Option 2: Claude via Google Cloud Vertex AI

- **What:** The same Claude models, billed through your existing Google Cloud / Firebase project.
- **Pros:** One bill, one project, no API key (uses the Cloud Function's service account). **Google Cloud startup credits can cover it.** The Google for Startups AI tier lists extra credit usable only on partner models such as Claude in Model Garden.
- **Check:** which regions host Claude. At last check, Claude on Vertex was **not listed in Mumbai (asia-south1)**. The nearest listed regions were Singapore (asia-southeast1) and Taiwan, plus a `global` endpoint. Your Functions stay in Mumbai; only the model call leaves India. Also check Vertex's own pricing for Claude, which can differ from Anthropic's.

### Option 3: Gemini via Vertex AI or Firebase AI Logic

- **What:** Google's own models, in the same Firebase project.
- **Pros:** Some Gemini models **can run in Mumbai (asia-south1)**, which is the simplest answer if you want data to stay in India. Covered by the main Google startup credits. Good Indic language support.
- **Check:** quality on your evaluation set, especially subtle risk messages. Check how it handles self-harm content (does it refuse or over-block?) and what its data-use terms say.

### Option 4: An Indian provider (for example Sarvam AI)

- **What:** Models built for Indian languages, hosted in India.
- **Pros:** Possibly the best Hindi, Marathi and code-mixed understanding. Data stays in India.
- **Check:** whether they offer a classification or structured-output API, uptime and support, data-retention terms, and pricing. Treat this as an option to evaluate, not a known fit.

### Option 5: Other hosted models (OpenAI, AWS Bedrock, Azure)

- Worth one quick look for completeness. Bedrock and Azure would add a second cloud account, which is extra work for a solo founder already on Firebase. Only pursue one if it clearly wins the evaluation or comes with credits.

### Option 6: Open model you host yourself (for example Gemma or Llama)

- **Pros:** Full control, data never leaves your project.
- **Cons:** You run and pay for a GPU server, handle updates and security, and quality on subtle risk is usually lower. **Not recommended for a solo, non-developer founder at this stage.** Keep it as a long-term option.

### Quick comparison (fill in as you test)

| | Claude direct | Claude on Vertex | Gemini on Vertex | Indian provider | Other |
|---|---|---|---|---|---|
| Runs in India? | No (global/US) | No (Singapore/global at last check) | Yes, some models | Likely yes | ? |
| Covered by credits you have? | | | | | |
| Recall on high-risk set (%) | | | | | |
| False alarms on safe set (%) | | | | | |
| Hindi / Marathi / Hinglish quality (1–5) | | | | | |
| Refuses to analyse self-harm text? | | | | | |
| Median latency (s) | | | | | |
| Cost per 1,000 messages (₹) | | | | | |
| No-training guarantee? | | | | | |
| Retention period | | | | | |
| Signed data processing terms available? | | | | | |

---

## 5. Paying for it: credits and real costs

### 5.1 You may not need credits at all

A rough estimate for triage with a small, fast model (for example Claude Haiku 5.5 at $0.10 / $0.50 per million tokens):

- One triage call ≈ 1,500 input tokens (instructions + message + a little context) + 100 output tokens
- ≈ $0.0002 per message
- **10,000 messages a month ≈ $2 a month** (about ₹170)

Summaries and drafts with a mid-size model (for example Sonnet 5.5) at about 4,000 input + 400 output tokens ≈ $0.012 each. **1,000 a month ≈ $12** (about ₹1,000).

So the early-stage AI bill is likely smaller than your phone bill. Credits are nice to have, not a blocker. Re-check prices before deciding.

### 5.2 Credit programs to check, one at a time

| Program | Covers | Notes to verify |
|---|---|---|
| **Anthropic startup program** (claude.com/programs/startups) | Claude API | You found nothing open when you checked. Third-party sites describe tiers, often tied to VC or accelerator partners. Check again every couple of months, or ask any accelerator you join whether it partners with Anthropic |
| **Google for Startups Cloud Program** (cloud.google.com/startup) | Firebase, Cloud Functions, Gemini, and (AI tier) partner models like Claude on Vertex | Probably the best fit, because you're already on Firebase. The AI tier asks for seed to Series A funding. Check the lower "Start" tier, which is usually open to unfunded founders |
| **Google for Startups Accelerator: India** | Credits + mentoring | Yearly cohort, equity-free |
| **Startup India / DPIIT recognition** | Not credits itself, but many programs ask for it | Free to apply. Makes you eligible for more |
| **AWS Activate, Microsoft for Startups** | Bedrock / Azure models | Only useful if you choose those clouds |
| **Mental-health or social-impact grants** (CSR funds, foundations) | Cash | Peer support with a safety focus is a good fit for impact funding |

---

## 6. Checklist for each provider

Copy this for every provider you test.

**Safety**
- [ ] Does it analyse self-harm and suicide text calmly, without refusing?
- [ ] Can it return strict JSON (structured output) so the code can't misread the answer?
- [ ] What happens on refusal or error? Can the code detect it reliably?

**Privacy and legal (India DPDP Act, 2023, and its rules)**
- [ ] Written guarantee that your data isn't used to train their models
- [ ] How long they keep prompts and responses. Is zero or short retention available?
- [ ] Where the processing happens (country, region)
- [ ] Data processing agreement (DPA) you can accept as a small business
- [ ] Mental-health conversations are sensitive. Get a short opinion from a lawyer familiar with DPDP before going live, especially on cross-border processing and consent wording
- [ ] Update the consent text in the app ("an automated system helps flag messages that may need urgent care") and bump `consentVersion`

**Practical**
- [ ] Price per million tokens (input and output), in your billing currency
- [ ] Median and slowest response time from Mumbai
- [ ] Rate limits on a new account
- [ ] Official Node/TypeScript SDK
- [ ] Works from Cloud Functions with a key in Secret Manager, or with no key (Vertex)

---

## 7. How to evaluate: the core of your free-time work

You can't pick a provider by reading websites. Build a small test set once, then run every option against it.

### 7.1 Build the test set (do this first, 2–3 sessions)

Write **synthetic** messages. **Never use real users' messages** for testing unless they've agreed to it.

Aim for about **200 messages**, in a spreadsheet with columns `id, text, language, group, expected_level, notes`:

| Bucket | How many | Examples of what to include |
|---|---|---|
| Imminent risk | 30 | Clear plans, goodbyes, "I've taken pills" |
| High risk, indirect | 40 | "Everyone would be better off without me", "I don't see the point anymore", postpartum "I'm scared of what I might do" |
| Medium (distress, no risk signs) | 40 | Grief, exhaustion, loneliness, exam or job stress |
| Safe / everyday | 50 | "Had a nice walk today", "Thanks for yesterday" |
| Tricky false alarms | 20 | "This traffic is killing me", "I'm dying to see my grandson", song lyrics, jokes |
| Abuse or danger from others | 20 | Domestic violence, elder neglect |

Mix languages throughout: English, Hindi (Devanagari), Marathi (Devanagari), **Hinglish and Romanized Marathi** (how people actually type), and some spelling mistakes. Include senior-style and mother-specific phrasing.

Mark each message's expected level yourself. If possible, ask one trusted mental-health professional to review the high-risk labels.

### 7.2 What to measure

| Metric | Target | Why |
|---|---|---|
| **Recall on imminent + high** (keywords + AI combined) | As close to 100% as possible. Every miss gets written down and studied | Missing a crisis is the worst failure |
| Extra catches by AI over keywords alone | The higher the better | This is the reason to add AI at all |
| False alarms on safe + tricky messages | Low enough that you don't start ignoring alerts (for example under 10%) | Alert fatigue makes you slower on real cases |
| Refusal rate | 0% | A refusal is a silent miss |
| Median latency | Under 5 seconds | You want the alert quickly |
| Cost per 1,000 messages | Record it | For budgeting |

### 7.3 How to run it

1. **By hand first (no code):** paste 20 messages from the set into each provider's free playground or console, with the same instructions. Note gut reactions: tone, refusals, language handling. This takes one evening per provider and rules out bad fits early.
2. **Scripted (with help):** a small script in `functions/scripts/eval-triage.ts` reads the spreadsheet, calls each provider through the `TriageProvider` interface, and writes a results sheet. Ask Claude Code to build this as its own task when you're ready.
3. **Compare** in the table in section 4. Run each provider twice to check its answers are consistent.
4. **Write the decision** in section 9 with the reason.

### 7.4 The triage instructions (first draft to test with)

Keep this in code as a versioned constant (`promptVersion`) so results can be traced back:

```
You are a safety screener for a peer-support chat app in India. A volunteer
supporter reads every message. Your only job is to help the supporter notice
messages that may need urgent care.

Read the user's latest message (and the earlier messages, if given) and return
JSON with:
- level: one of none, low, medium, high, imminent
  - imminent: signs of a current plan, means, or an attempt in progress
  - high: thoughts of suicide or self-harm, wishing to die, or being in danger
    from someone else, even if said indirectly
  - medium: strong distress or hopelessness without risk signs
  - low: everyday sadness or stress
  - none: no distress
- categories: any of self_harm, suicide, harm_to_others, abuse, postpartum,
  hopelessness, substance, medical, other
- reason: one short sentence in English for the supporter. Do not quote the
  message.

Messages may be in English, Hindi, Marathi, or mixed and Romanized
(Hinglish). Treat figures of speech ("this traffic is killing me") as such.
When unsure between two levels, choose the higher one.
```

---

## 8. Step-by-step plan: what to do next

Work top to bottom. Each step is small enough for one or two free evenings. Tick it when done.

**Stage 0: Decisions and paperwork**
- [ ] 0.1 Decide the open question in section 3.1 (does an AI "imminent" also show the user the crisis screen?) and write the answer into `docs/SAFETY.md`
- [ ] 0.2 Apply for DPIIT / Startup India recognition (helps with every credit program)
- [ ] 0.3 Apply to the Google for Startups Cloud Program (the tier you qualify for)
- [ ] 0.4 Note the date you last checked the Anthropic startup program. Re-check in 2–3 months

**Stage 1: Test set**
- [ ] 1.1 Create the 200-message spreadsheet (section 7.1). Keep it out of the public repo if it has anything personal; synthetic only
- [ ] 1.2 Get a professional to review the high-risk labels, if possible

**Stage 2: Hand testing (one provider per evening)**
- [ ] 2.1 Claude (Anthropic console): Haiku 5.5, Sonnet 5.5, Opus 5.5 on 20 messages
- [ ] 2.2 Gemini (Google AI Studio or Vertex console) on the same 20
- [ ] 2.3 One Indian-language provider on the same 20
- [ ] 2.4 Drop any option that refuses, misreads Hinglish badly, or can't return clean JSON

**Stage 3: Privacy check for the 2–3 survivors**
- [ ] 3.1 Fill in the privacy part of the section 6 checklist for each
- [ ] 3.2 Short consult with a lawyer on DPDP, cross-border processing and consent wording

**Stage 4: Scripted evaluation** *(ask Claude Code to build the script as a ROADMAP task)*
- [ ] 4.1 Build `functions/scripts/eval-triage.ts` and the provider interface
- [ ] 4.2 Run the full set on each survivor, twice
- [ ] 4.3 Fill in the comparison table, pick a provider and model, and record the decision in section 9

**Stage 5: Build triage (Phase 2 ROADMAP tasks)**
- [ ] 5.1 `triage` collection + Firestore rules + rules tests
- [ ] 5.2 Provider implementation behind `TriageProvider`, key in Secret Manager (or Vertex service account)
- [ ] 5.3 Wire into `onMessageCreated` after the keyword check, with timeout and fail-safe. Unit tests prove: keywords still fire when AI is down; AI can raise but never lower a level; no message text in logs
- [ ] 5.4 Admin dashboard: show "flagged by AI" with the reason, so you can see why
- [ ] 5.5 Update consent text and `consentVersion` (all three languages)
- [ ] 5.6 Run in **shadow mode** for 2–4 weeks: AI results are saved but don't create alerts. Compare with what you'd have done. Then switch on

**Stage 6: Summaries and drafts (later)**
- [ ] 6.1 Conversation summary button in the admin view (admin-only callable function)
- [ ] 6.2 Draft-reply button. The draft opens in your reply box for editing and is never auto-sent
- [ ] 6.3 Re-run the evaluation whenever you change provider, model or instructions

---

## 9. Decision log

Record each decision with the date and the reason, so future you (or a helper) knows why.

| Date | Decision | Options considered | Why |
|---|---|---|---|
| | | | |

---

## 10. Sources checked (November 2026)

Re-verify these, since terms change.

- Anthropic model pricing: Anthropic API documentation (platform.claude.com)
- Anthropic startup program: claude.com/programs/startups (third-party summaries disagree on amounts and eligibility)
- Google for Startups Cloud Program, AI tier: cloud.google.com/startup/ai
- Vertex AI generative AI data residency and regions: cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency
- India Digital Personal Data Protection Act, 2023, and its rules: meity.gov.in
