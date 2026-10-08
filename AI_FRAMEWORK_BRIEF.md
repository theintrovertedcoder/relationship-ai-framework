# Mole's AI framework: brief for the design conversation

*5 Oct 2026. Written to start a separate conversation in which Haziq and Claude
design a proper AI framework for Mole: how Mole uses AI to solve memory, and
the rules every AI feature follows. When the framework is finished, it comes
back to this repository as `AI_FRAMEWORK.md`, next to this brief in docs, and
is implemented here.
Everything below is how the code works today, checked against the code on this
date. Nothing here is a decision yet.*

**How to use this:** attach or paste this whole file at the start of the new
conversation. Section 9 is the agenda; the deliverable is described in
section 10.

---

## 1 · Mole, and the problem AI is meant to solve

Mole remembers the people who matter and tells you when to reach out. People
meet more people than they can remember: who someone was, what was said, what
they care about, when to follow up. Mole captures a person in seconds (scan a
business card, tap an NFC card, save a contact), keeps what you learn about
them, and brings the right person back at the right moment.

It is one brand across five products: the **Mole app** (Vibe, Cards, Signals,
Persona) with the public pages anyone opens from a card or QR code; **Loop**,
the B2B product (organisations, their members, and two add-ons, **Events** and
**Spaces**); **Mole Bingo**, human bingo for events; **the den**, the staff
dashboard; and the **marketing sites**. Users are 18 or over. The company is in
Malaysia and Singapore.

**The canonical brand, voice included, is the one Mole design system:**
https://claude.ai/artifact/4yHci7Z3FuU7N2dERYd77p. Where anything here
disagrees with it, it wins. This brief covers the AI in the Mole app, Loop and
the den, which is where the AI is today; the framework should also say whether
and how it extends to Mole Bingo and the marketing sites.

AI is how the remembering and the "right moment" are meant to happen. Today it
is a set of separate features rather than one system.

## 2 · What Mole holds about people

What the AI could draw on, as the data exists today:

| Kind | What it holds |
|---|---|
| **Contacts** | Name, company, role, email, phone, notes, tags, free-text context, photos, the scanned card image, and a history of interaction summaries |
| **Meetings** | Meetups on a calendar, with a title and the contact |
| **Reminders** | Follow-ups the person set, and kinds of reminder |
| **The person's own profile** ("digital twin") | Professional goals and challenges, professional and personal interests, values, how they like to communicate, networking style, and a list of intents (what they are looking for). Private to them |
| **Their public card** | What strangers see, who viewed it, who saved it |
| **Signals** | News and changes matched to a contact and to the person's intents |
| **Loop** | Organisations, members, events with attendees and check-ins, buildings with visitors, forms and their answers |

## 3 · Every AI feature today

All of it runs on Google Gemini. Every call from the app goes through one server
function, `ai_proxy`. Scheduled jobs and embeddings call Google from their own
functions.

| Feature | When it runs | What is sent to Google | Model |
|---|---|---|---|
| **Card scanning** | The person photographs a business card | The photo. The prompt says never invent a field and never guess an email | `gemini-flash-latest` by default |
| **Ask Mole search** | The person asks a plain-English question about their contacts | The question (and an optional image), plus **every contact they have**: name, company, role, notes, tags, context and history summaries, in one request | Same |
| **Ask Mole chat** | A support question (built, switched off) | The question and the published FAQ entries only. It answers from the FAQs or says it can't | Same |
| **Signals, on demand** | The person asks for a signal about a contact | The contact's name, company and role, the person's intents, and their profile paragraph (goals, interests, style; capped at 800 characters) | Same |
| **Signals, scheduled** | Daily, for up to 200 people | The same kind of input, from `generate_signals` | Same |
| **Meeting agenda** | Before a meeting | Meeting title, contact name, the meeting's context, and the person's profile paragraph | Same |
| **Matching** | When contacts and news items are saved | Contact and news text, turned into embeddings for similarity matching (`embed_contact`, `embed_news`, `match_contacts`) | `text-embedding-004` |

Other model names appear in the configuration (`gemini-flash`, `gemini-flash-lite`,
`gemini-pro`). Which feature uses which is set per feature in the den.

**Not AI, but part of the same system:** `signals-engine/` is a worker that
reads news feeds and matches items to contacts in four tiers. It is written and
tested and has never run in production (YOUR_TURN B25).

## 4 · How it is built today: topology, architecture, infrastructure, layers

### Topology

```
  Phone / browser (React app)                         Mole staff (the den)
        │  signed-in user token                              │
        ▼                                                    ▼
  Cloudflare (DNS, TLS, crawler worker) ──► Railway: the web app (static files only; no AI here)
        │
        ▼  HTTPS + user token
  ┌───────────────────────────── Supabase ──────────────────────────────┐
  │  Edge functions (Deno)                       Postgres                 │
  │   ai_proxy ──── checks: config, flags,       ├ contacts (+ embedding  │
  │     │          Pro, limits; reads persona    │   vector(768), ivfflat)│
  │     │          from user_settings            ├ news_items (+ vector)  │
  │   embed_contact ◄── the app, or a DB webhook ├ ai_service_config      │
  │   embed_news    ◄── signals worker asks      ├ ai_call_log, limits    │
  │   match_contacts ─► match_*_semantic RPCs    ├ user_settings (persona)│
  │   generate_signals ◄─┐                        └ Vault: Gemini key,     │
  │                      │ pg_cron + pg_net,          cron secret          │
  │                      └─ secret from Vault                              │
  └──────┬────────────────────────────────────────────────────────────────┘
         ▼  HTTPS, API key
   Google Gemini API  (generateContent: flash / flash-lite / pro;
                       embeddings: text-embedding-004)

  Railway: signals-engine worker (written, never run) ── polls news feeds,
     writes news_items, asks embed_news to embed them, applies a company stoplist
```

### Software layers, as they are

| Layer | What is there | Where |
|---|---|---|
| **Interface** | Card scanner, Ask Mole (search, chat), Signals tab, meeting briefing, the den's AI Control screen | `components/`, `pages/` |
| **Client AI service** | One module the screens call; in sample-data mode it answers without calling anything | `services/aiService.ts`, `services/geminiService.ts`, `lib/edgeFunctions.ts` |
| **Gateway** | `ai_proxy`: authenticates the person, reads per-feature config, enforces Pro and limits, builds the prompt, calls the model, checks the answer's shape, logs the cost | `supabase/functions/ai_proxy/` |
| **Prompts** | Written inline in each function's code, one per feature | Inside the functions |
| **Background AI** | Scheduled Signals, embeddings, semantic matching | `generate_signals`, `embed_contact`, `embed_news`, `match_contacts` |
| **Memory store** | Contacts, history, meetings, reminders, the person's profile, news items, vectors | Supabase Postgres with pgvector |
| **Model provider** | Google Gemini, called directly with an API key; each function carries its own copy of the connection code | `generativelanguage.googleapis.com` |
| **Control and observability** | Per-feature config, call log with cost, limits, feature flags, Sentry for errors | `ai_service_config`, `ai_call_log`, den → AI Control, Sentry |

### Infrastructure

| Piece | Runs on | Role in AI |
|---|---|---|
| Web app | Railway (static server behind Cloudflare) | None; sends requests only |
| Edge functions | Supabase (Deno) | Every AI call |
| Database and vectors | Supabase Postgres + pgvector | Memory, embeddings, similarity search, config, logs |
| Secrets | Supabase Vault and function secrets | Gemini key, cron secret |
| Scheduler | pg_cron + pg_net inside Postgres | Starts scheduled Signals and other sweeps |
| News worker | Railway (planned; not running) | Feeds Signals |
| Model | Google Gemini API | Text, vision, embeddings |
| Monitoring | Sentry (errors), the den (spend and calls) | No AI-quality monitoring yet |

### The four paths an AI call takes today

1. **On request:** app → `ai_proxy` (person's token) → Gemini → answer checked
   → cost logged → app.
2. **On save:** a contact is saved → `embed_contact` (called by the app, or by
   a database webhook if one is set up) →
   Gemini embeddings → vector stored on the contact.
3. **On schedule:** pg_cron → `generate_signals` (cron secret) → Gemini →
   signals stored → seen in the app.
4. **From the world:** news feeds → signals worker → `news_items` →
   `embed_news` → vectors → `match_contacts` → candidate matches for Signals.

## 5 · The controls that already exist

- **The key never reaches a phone.** The Gemini key is stored in Vault on the
  server and set from the den.
- **Per-feature settings** (`ai_service_config`): model, maximum length,
  temperature, a cost cap, and on/off for each feature. They're changed in
  the den on the **AI Control** screen, which also shows spend and has a test
  panel.
- **Limits:** each person gets 200 calls a day. Each feature also has its
  own daily, weekly and monthly limits, so one feature running out doesn't
  stop the others.
- **Cost:** each call through `ai_proxy` is logged with its cost
  (`ai_call_log`).
- **Pro:** some features are for Pro only, enforced once the paywall flag is
  on.
- **Consent:** Signals is off for everyone until the person turns it on.
- **Personal context:** the person's profile is read from their private
  settings on the server, never from the request, so the app can't claim to
  be someone else. It's cleaned so it can't break the prompt.
- **Images:** limited to 8 MB.

## 6 · What is already written down

These documents in the repository go deeper on parts of this:

| Document | What it covers |
|---|---|
| `SIGNALS_ARCHITECTURE.md` | The long-term design for Signals: observe → detect change → match against intent → only then speak |
| `SIGNALS_V1_BRIEF.md` | Version 1, which works the other way round: a news item arrives and Mole asks who cares, so cost grows with the news rather than with users |
| `ASK_MOLE_CHAT.md` | How Ask Mole chat is switched on, and what it will and won't say |
| `ASK_MOLE_EVAL.md` | A scored test set for Ask Mole search and chat. It's the only measured quality number for any AI feature |
| `CAPACITY_AND_COST.md` | What AI costs at 1k, 10k and 100k users, and what breaks first |
| `PRIVACY_INVENTORY.md` | What personal data Mole holds, and Google listed as a processor |

## 7 · What is missing, or wrong, today

1. **No single set of rules.** Nothing says what personal data may go to
   Google, what never may, who must opt in, how a feature is measured, or how
   it's switched off in an emergency. Each feature made its own choices.
2. **Ask Mole search sends a person's whole contact book, notes included,
   with every question.** That's the largest data exposure, and the cost
   grows with the size of the contact book. Embeddings already exist and could
   pick a shortlist first.
3. **Quality is measured for Ask Mole only.** Card-scanning accuracy is
   unknown until 30 real cards are photographed (YOUR_TURN B6), and nothing
   measures Signals or the agenda.
4. **Prompts live in the code,** so changing one means a code change and a
   deploy. The den can change the model and settings but not the words.
5. **Mixed models,** and no rule for which job gets which.
6. **Five copies of the Gemini connection code,** because the dashboard
   can't share files between functions. A fix has to be made five times.
7. **Untrusted text goes into prompts.** A scanned card, a news item or a
   contact's notes can contain instructions, and nothing treats them as data
   only.
8. **Nothing explains itself to the person.** A signal doesn't say why it
   appeared, or which source it came from, in a consistent way.
9. **No rule for Loop.** It's undecided whether AI may ever read an
   organisation's data (members, attendees, visitors), and on whose authority.

## 8 · Rules the framework has to respect

- **Personal data:** the Privacy Policy names Google as a processor and
  promises users are 18 or over. Any new processor, or any new kind of data
  sent, has to go into the policy and to the lawyer (YOUR_TURN B2).
- **Keys stay on the server:** Courier, Stripe and Gemini keys are
  server-only secrets. Never on the phone, never in the app bundle.
- **Admin and user stay separate:** staff powers are granted in Team &
  Roles only. Mole Admin functions never appear in user management. Staff
  actions are logged; staff *reads* are limited by role but not yet logged.
- **Consent before profiling:** Signals is off until the person turns it on.
  Any feature that profiles a person, or watches the world on their behalf,
  should be the same.
- **Never invent:** card scanning already must not invent fields. The same
  standard should apply everywhere: a missing fact is left blank, not
  guessed.
- **Retention:** form answers and guests are deleted on schedule
  (migration 128). Whatever the AI derives needs its own retention rule.
- **Tone of voice is canon, from the design system:** anything the AI writes
  follows its two registers. **The product voice** (neutral, plain, calm,
  specific, no emoji) is for the app, the den, Loop, hosts and every system
  message. **Sunny's voice** (warm, short, first person, an emoji only inside a
  line Sunny says) is for Bingo guests, emails and the marketing sites. In
  both: write to one person as "you", sentence case, British spelling, say
  what happened and what to do next, never show a raw error, and never guess
  a cause the error doesn't show. A refusal is a sentence the person can act
  on.
- **Visible to Haziq in a browser:** any check, switch or number the
  framework needs must be a page he can open, not a command. In practice,
  that's the den.

## 9 · The agenda: questions the framework should answer

**A. What "AI solves memory" means for Mole**
- Which moments AI serves: capture, enrich, recall, remind, prepare, reflect?
- Which moments it must never act in on its own? For example, sending a
  message for the person, or changing a contact without asking.
- What does success look like for each moment, in words a user would use?

**B. The memory model**
- What is one "memory": a contact, a fact, an interaction, a signal?
- Where does each fact come from (the person, a card, AI, the news), and how
  sure is Mole about it?
- How does a person see, correct and delete what Mole remembers or worked
  out? How do old facts fade?

- Which register each AI feature speaks in (product voice or Sunny's voice),
  and how that is tested.

**C. Data rules**
- What may go to an AI provider, and what never may?
- Minimise by default: shortlist before sending, send fields rather than
  records (see §7.2).
- Sensitive fields, provider retention and training opt-outs, and where
  anything could stay on the phone.

**D. Consent and control**
- Which features are on by default, and which need opt-in?
- How a person sees why something appeared, and its source.
- One off switch per feature, and one for all of AI.

**E. Quality**
- A scored test set for every feature, with a bar it must clear before it
  ships or a prompt or model changes.
- Who reviews failures.

**F. Models and providers**
- Which model does which job, and why.
- Pinning model versions.
- A fallback when the provider is down.
- Whether to stay on one provider.

**G. Prompts**
- Where they live, how they are versioned, whether staff can edit them in
  the den, and how a change is tested before it's published.

**H. Cost and limits**
- Budgets per person, per feature and per plan (Free and Pro).
- What happens at the limit, in words the person sees.

**I. Safety and failure**
- Untrusted text in prompts (§7.7).
- Wrong answers, outages and abuse.
- What the person sees when AI fails.

**J. Loop and organisations**
- Whether AI may use organisation data (members, attendees, visitors), for
  whom, and with whose consent.

**K. Topology, architecture and layers (the target)**
- **Topology:** which parts talk to which, and across which trust
  boundaries: the phone, the gateway, background jobs, the memory store, the
  model provider, the news worker, the den.
- **Software architecture:** a layer model, with what each layer owns and
  what it must not do. For example: interface → client AI service → gateway →
  policy (consent, limits, data rules) → prompt and context builder → model
  adapter → memory store, with evaluation and observability across all of
  them.
- **Infrastructure:** where each layer runs (Supabase functions, Postgres
  and pgvector, Railway workers, the provider), and what changes with scale
  (1k, 10k, 100k users, per `CAPACITY_AND_COST.md`). Regions and data
  residency for Malaysian and Singaporean users. Secrets, scheduling, queues
  and retries.
- **The path from today to the target:** what moves, in what order, without
  breaking the features that work now.

**L. Watching it run**
- What is logged, without personal data.
- What shows on the den's AI Control screen.
- What raises an alert.

## 10 · What to bring back

One document, `AI_FRAMEWORK.md`, to sit next to this brief in docs:

1. **Principles:** a short list every AI feature follows.
2. **Topology, architecture and layers:** the target topology diagram, the
   layer model with each layer's job and boundaries, the infrastructure each
   runs on, and the migration path from today's shape (section 4).
3. **The memory model** (section 9B).
4. **One page per feature, existing and planned:**
   - its job, and when it runs;
   - data in, data out, and what is never sent;
   - model, consent, limits;
   - how quality is measured and the bar it must clear;
   - the off switch, and what the person sees when it fails.
5. **Decisions, numbered,** so each can be confirmed or changed one at a time
   (as D19–D33 were for the build).
6. **Open questions** that need Haziq or a lawyer.

Back here, it gets mapped against the code feature by feature: what already
complies, what changes, and what is new. Each gap becomes a task with a test
that fails until it's done, and it's built here.
