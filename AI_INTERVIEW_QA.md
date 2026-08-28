# AI Tools & Exposure — Real Interview Questions & Answers

> This file is a personal log of actual questions asked to me by interviewers in real interviews about AI tool exposure — coding assistants, AI-assisted automation, and how AI fits into a DevOps workflow — as opposed to questions tied to one specific infrastructure tool (which live in DOCKER_INTERVIEW_QA.md, KUBERNETES_INTERVIEW_QA.md, TERRAFORM_INTERVIEW_QA.md, AWS_INTERVIEW_QA.md, PYTHON_INTERVIEW_QA.md, DEVOPS_INTERVIEW_QA.md).
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Wipro | DevOps Engineer | Technical Round 1](#interview-1)
  - [Q1. Do you have any AI exposure? Which AI coding/automation tools have you used, and how — e.g. Cursor AI, GitHub Copilot, Claude Code?](#q1-do-you-have-any-ai-exposure-have-you-used-any-ai-tools-in-your-organization-have-you-implemented-ai-tools-in-your-current-or-recent-project-which-ai-codingautomation-tools-have-you-used-how-have-you-used-tools-such-as-cursor-ai-github-copilot-or-cloud-code)
  - [Q2. Do you know about prompts, tokens, and how these AI tools work? Explain.](#q2-do-you-know-about-prompts-tokens-and-how-these-ai-tools-work-explain)
  - [Q3. What is RAG? How does RAG work?](#q3-what-is-rag-how-does-rag-work)
  - [Q4. What is MCP (Model Context Protocol) and why is it used?](#q4-what-is-mcp-protocol-what-is-model-context-protocol-and-why-is-it-used)

---

## Interview #1

**Company:** Wipro
**Date:** 23-08-2026
**Role Applied For:** DevOps Engineer
**Round:** Technical Round 1
**Interviewer Level:** Not specified

---

### Questions Asked

#### Q1. Do you have any AI exposure? Have you used any AI tools in your organization? Have you implemented AI tools in your current or recent project? Which AI coding/automation tools have you used? How have you used tools such as Cursor AI, GitHub Copilot, or Claude Code?

**Answer:**

Yes — and I'd separate this into two genuinely different things, because the question bundles them together: **using AI tools to help build/operate infrastructure and automation** (which I do daily) versus **shipping AI as a feature of a product** (a different, much bigger claim I wouldn't want to overstate). Almost everything I'd point to is the first category — AI as a tool in my own workflow, not something I've deployed as an AI product for end users.

---

**The tools, and what each is actually good for**

| Tool | What it is | How I actually use it |
|---|---|---|
| **GitHub Copilot** | Inline, in-editor code completion | Fast boilerplate — repetitive Terraform resource blocks, Kubernetes manifest scaffolding, test scaffolding — accepted or rejected line by line as I type |
| **Cursor** | An AI-native editor with chat, multi-file edits, and codebase-aware context | Larger, more contained refactors within a project — "rename this pattern across these files," "explain what this Terraform module does" — where I want a conversational back-and-forth grounded in the actual repo, not just one line at a time |
| **Claude Code** | An agentic CLI/terminal tool that can read, edit, and run commands across a whole repository, including git operations | Multi-step, repo-wide tasks — the kind where I'd otherwise be doing a lot of manual file-by-file work myself: auditing a repo for structural consistency, writing and organizing documentation, running and interpreting `terraform plan`/test output as part of the task, then committing the result |
| **ChatGPT/Claude (chat, not IDE-integrated)** | General-purpose LLM chat | Quick, ad-hoc explanation of an unfamiliar error message or stack trace, drafting a first pass at a runbook or incident postmortem structure, sanity-checking an approach before I implement it |

---

**A concrete, verifiable example — this actual repository**

Rather than describe a hypothetical, I'd point to something directly demonstrable: this interview question-and-answer repository itself was built using **Claude Code** as an agentic pair — not autocomplete, but a genuine multi-step collaborator across an entire session: writing detailed technical answers, auditing the repo's file structure and catching a real bug (a scrambled Table of Contents across three files, where a later interview's heading had gotten inserted before an earlier interview's own question list), reorganizing content, and handling the git commit/push workflow at my direction. That's a real, ongoing example of AI-assisted work I can actually show, not just describe — the interviewer can look at the repo's own commit history.

**In the CloudCart project specifically**, the day-to-day pattern looks similar but smaller in scope: Copilot for fast Terraform/YAML boilerplate while writing a new module, Cursor when I need to understand or refactor something spanning several files at once (e.g., "everywhere this naming convention is used, apply the new validation pattern from the Terraform interview's Q5"), and Claude Code for genuinely multi-step tasks — auditing a set of Dockerfiles for a consistent multi-stage pattern, or drafting a first version of a runbook from a set of past incident notes, then reviewing and correcting it myself before it goes anywhere real.

---

**The principle I'd state explicitly — trust but verify, especially for infrastructure**

This is the point I'd make sure comes across, because it's the difference between "I use AI tools" and actually understanding their risk profile in a DevOps context: **AI-generated infrastructure code gets the exact same review discipline as a human-written PR — never more trust, specifically less by default.** A hallucinated IAM policy, an overly broad security group, or a subtly wrong Terraform resource argument is a much bigger blast radius than a wrong line of application code, and LLMs will confidently produce plausible-looking, syntactically valid, semantically wrong infrastructure code with no warning sign that it's wrong. Concretely, that means:
- AI-suggested Terraform still goes through `terraform plan`, `tflint`, `checkov`/`tfsec`, and a human PR review — exactly the pipeline from the Terraform interview's Q15 — regardless of who or what wrote the diff.
- I don't accept an AI suggestion for anything security-sensitive (IAM policies, network rules, secrets handling) without reading and understanding every line myself, the same way I wouldn't merge a junior engineer's PR in that area without a careful review.
- For genuinely critical/production changes, AI is used for drafting and acceleration, not as the final decision-maker — the same manual-approval-gate-before-PROD principle that shows up throughout the CI/CD design (DevOps Q4, Interview #1) applies to AI-assisted changes too, not just human-written ones.

---

**Real-world example — CloudCart**

Early on, an AI coding assistant suggested a Terraform security group rule that looked reasonable at a glance — correct syntax, plausible CIDR — but on review turned out to open a port more broadly than intended, because the suggestion had generalized from a different, less sensitive resource elsewhere in the same repo rather than actually reasoning about this specific resource's requirements. It was caught in the normal PR review process, specifically because the team's rule is that AI-authored and human-authored infrastructure changes go through identical review — nothing skipped a step because "the AI wrote it and it looked fine." That's the incident I'd point to if asked for a downside: not that AI tools are unreliable to the point of being unusable, but that they need the same skepticism and process as any other source of a diff, not less.

---

**Complete thought process — how I approach this in the interview**

```
Two different claims bundled in this question — separate them:

1. AI tools I use to DO my own work faster
   → Copilot: inline completion, fast boilerplate
   → Cursor: multi-file, codebase-aware refactors
   → Claude Code: agentic, multi-step, repo-wide tasks — including
     THIS repo as a concrete, checkable example
   → Chat LLMs: ad-hoc explanation, drafting, sanity-checking

2. AI as a shipped product feature — NOT overclaiming this, since
   most of my exposure is category 1, not category 2

The point that actually shows judgment, not just tool familiarity:
  → AI-generated infra code gets the SAME review pipeline as
    human-written code — plan, lint, security scan, PR review —
    never skipped because "the AI wrote it"
  → Concrete example of this mattering: an AI-suggested security
    group rule that looked fine but was too broad, caught in normal
    review specifically because nothing was exempted from review
```

---

**Summary (what to say if time is short):**

*"Yes — though I'd split it into two things this question bundles together: AI tools I use to do my own work faster, versus AI shipped as a product feature, and almost everything I'd point to is the first category. Day to day, that's GitHub Copilot for fast, inline boilerplate — repetitive Terraform blocks, Kubernetes manifests — Cursor when I need a codebase-aware refactor spanning several files, and Claude Code for genuinely multi-step, repo-wide tasks, like auditing a project's structure or drafting documentation and then handling the review-and-commit workflow. Actually, this interview question bank itself is a concrete example — it was built with Claude Code as an agentic collaborator across the session, including catching and fixing a real structural bug in the repo, which the interviewer can look at directly rather than take my word for. The principle I'd emphasize, though, is that AI-generated infrastructure code gets the exact same review discipline as human-written code — plan, lint, security scanning, PR review — never less, because a wrong IAM policy or security group rule from a confidently-wrong AI suggestion has a much bigger blast radius than a wrong line of application code. I've actually seen that exact failure mode — an AI-suggested security group rule that looked fine at a glance but was broader than intended — caught in normal review specifically because nothing gets exempted from review just because AI wrote it."*

---

#### Q2. Do you know about prompts, tokens, and how these AI tools work? Explain.

**Answer:**

I'd answer this at the level a DevOps engineer actually needs — practitioner-level understanding of the mechanics that affect cost, latency, and output quality, not claiming deep ML-researcher knowledge of transformer internals I don't have. That distinction matters and I'd say it explicitly.

---

**Prompt — the actual input the model sees**

A prompt is the text sent to the model to get a response — but in any real tool (Copilot, Cursor, Claude Code, or an API call I've written myself), it's rarely just "what the user typed." It's assembled from several parts:
- A **system prompt** — instructions set by the tool itself, defining behavior/role/constraints, not visible to the end user
- The **user's actual message** — what I typed
- **Injected context** — for a coding tool specifically, this is the part that matters most: open files, relevant search results from the codebase, a git diff, or file contents the tool decided were relevant to the request

That last point is the one I'd volunteer as a practitioner insight, not textbook trivia: **how well an AI coding tool performs on a large codebase depends heavily on how well it selects what context to inject**, not just on the underlying model's raw capability. A tool that stuffs in too much irrelevant code wastes context budget and can actually make output worse — a real, known effect sometimes called "lost in the middle," where information buried in the middle of a very long prompt gets less effective attention than information near the start or end.

---

**Tokens — the actual unit the model processes**

Models don't process characters or whole words directly — they process **tokens**, produced by a **tokenizer** that breaks text into sub-word pieces. A rough rule of thumb for English: about **4 characters, or roughly ¾ of a word, per token** — so "tokenization" itself might split into two or three tokens, not one, depending on the tokenizer. This matters practically in a few concrete ways:

- **Context window** — the maximum number of tokens (input + output combined) a model can handle in a single request is measured in tokens, not words or characters. This is exactly why a coding tool working across a large codebase needs a large context window, and why it can't just dump an entire massive repository into one prompt regardless of window size — token budget is finite and has to be spent deliberately.
- **Cost** — API-based LLM usage (as opposed to a fixed-subscription IDE tool) is typically billed per token, input and output priced separately, usually per-million-token rates. If I were building an automation that calls an LLM API — say, summarizing an error log for a Slack alert — token count directly drives both cost and latency, so trimming the input to only what's relevant before sending it isn't just tidiness, it's a real design decision with a cost/latency consequence.
- **Output generation is also token-by-token** — the model generates one token at a time, each new token conditioned on everything before it (prompt plus every token generated so far), which is also why longer requested outputs take measurably longer and cost measurably more, not just more text for the same price.

---

**How the tools actually work, at the level worth knowing**

At a high level: these are large neural networks (the transformer architecture) trained on massive amounts of text to predict the next token given everything before it. At inference time — when I actually send a prompt — the model repeats that same prediction step over and over, each generated token appended to the context and fed back in, until it decides it's done (or hits a length limit). Two practically relevant knobs on top of that:

- **Temperature** — controls how deterministic vs. varied the output is. Lower temperature (near 0) gives more consistent, repeatable output for the same input; higher temperature adds more variety/creativity. This is a genuinely relevant design choice, not just trivia: for an automated pipeline where I need consistent, parseable output every time (e.g., a script relying on an LLM API to always return the same structured format), I'd want low temperature; for open-ended brainstorming or writing assistance, higher temperature is usually preferred.
- **Streaming** — most of these tools show output token-by-token as it's generated rather than waiting for the full response, which is why a coding assistant's response appears to "type itself out" rather than showing up all at once.

---

**Real-world example — CloudCart**

A small internal tool sends recent error-log excerpts to an LLM API to generate a short, human-readable one-line summary attached to a Slack alert, rather than the raw stack trace — genuinely useful when the on-call engineer needs to triage several alerts quickly. Two of the concepts above directly shaped how it's built: the input is deliberately **trimmed to only the relevant log lines** before being sent, rather than the full log file, both to control token cost on a service that fires fairly often and because a shorter, focused prompt produces a more reliable summary than a long one padded with irrelevant lines; and the API call uses a **low temperature** setting specifically because the summary feeds directly into an automated alert message — consistency run to run matters more there than creative variation would.

---

**Complete thought process — how I approach this in the interview**

```
Answer at the right altitude: practitioner mechanics that affect
cost/latency/quality, not claiming ML-researcher depth I don't have

Prompt:
  → Not just "what I typed" — system prompt + user message +
    injected context (open files, search results, diffs)
  → Context SELECTION quality matters as much as raw model
    capability for how well a coding tool performs on a large repo

Tokens:
  → Sub-word units, ~4 chars/¾ word per token in English
  → Context window = max tokens per request, input+output combined
  → API billing is per-token — directly shapes automation design
    decisions (trim input before sending, for cost AND quality)
  → Output is generated token-by-token, autoregressively

Practical knobs worth naming: temperature (deterministic vs varied
output — matters for automation needing consistent output) and
streaming (why responses appear to type themselves out)
```

---

**Summary (what to say if time is short):**

*"At a practitioner level, yes. A prompt isn't just what I type — in any real tool it's assembled from a system prompt, my actual message, and injected context like open files or search results from the codebase, and how well that context gets selected actually affects output quality as much as the underlying model's capability does. Tokens are the actual unit the model processes — roughly four characters or three-quarters of a word each in English — and that matters practically in three ways: the context window, which is the max tokens per request, input and output combined; cost, since API-based usage is typically billed per token, so trimming irrelevant input before sending it is a real design decision, not just tidiness; and generation itself, since the model produces output one token at a time, conditioned on everything before it, which is also why longer outputs cost and take measurably more. At a mechanical level, these are transformer-based models trained to predict the next token, and at inference time that same prediction step just repeats until the response is done — with temperature controlling how deterministic versus varied the output is, which matters if I'm calling an LLM API from an automation that needs consistent output every time versus something more open-ended. I'd apply that directly — a small internal tool I described earlier trims log input to only the relevant lines before sending it to an LLM API, both for cost and because a focused prompt produces a more reliable summary, and it runs at low temperature specifically because the output feeds an automated alert message where consistency matters more than variation."*

---

#### Q3. What is RAG? How does RAG work?

**Answer:**

RAG — **Retrieval-Augmented Generation** — is a technique for grounding an LLM's answer in specific, external documents retrieved at query time, rather than relying purely on whatever the model happened to learn during training. The core problem it solves: a model's training data is **frozen at some cutoff point** and generally doesn't include an organization's private, internal, or genuinely current information — runbooks, internal wikis, recent incident postmortems, this repo's own content. RAG bridges that gap by fetching relevant real documents and handing them to the model as part of the prompt, so the answer is generated **from that retrieved material**, not just from the model's pre-trained knowledge.

---

**How it actually works — two distinct phases**

**Phase 1 — Indexing (done ahead of time, whenever the source documents change)**
```
Source documents (wikis, runbooks, past incident postmortems, docs)
    ↓
Split into smaller CHUNKS (a paragraph or a few hundred tokens each —
whole documents are usually too large and too unfocused to embed
and retrieve well as a single unit)
    ↓
Each chunk → an embedding model → a numeric VECTOR representing
its meaning
    ↓
Vectors stored in a VECTOR DATABASE (Pinecone, Weaviate, pgvector,
FAISS, OpenSearch's vector support), each vector linked back to its
original chunk of text
```

**Phase 2 — Retrieval + Generation (happens live, per query)**
```
User's question
    ↓
Same embedding model turns the QUESTION into a vector too
    ↓
Vector database does a similarity search (nearest-neighbor lookup,
usually cosine similarity) → returns the top-K most relevant chunks
    ↓
Those retrieved chunks get inserted into the PROMPT as context,
alongside the original question
    ↓
The LLM generates its answer using both — grounded in the actual
retrieved text, not purely from what it learned during training
```

The key mechanical insight: **the model itself doesn't change at all.** RAG doesn't retrain or fine-tune anything — it controls what the model *sees* at the moment of answering, by retrieving the right context and putting it directly into the prompt (tying back to Q2's point that a prompt is rarely just "what the user typed" — retrieved context is exactly the kind of injected content that makes up the rest of it).

---

**RAG vs. fine-tuning — the comparison I'd volunteer, since it's the natural follow-up**

| | RAG | Fine-tuning |
|---|---|---|
| **What changes** | Nothing about the model — only what's retrieved and injected into the prompt | The model's own weights are updated |
| **Updating with new information** | Cheap and fast — just re-index the updated documents | Slow and expensive — requires retraining/re-running a training job |
| **Traceability** | Can cite exactly which source chunk an answer came from | Knowledge is baked into weights — no way to point to a specific source |
| **Best for** | Answering from a large, frequently-changing body of documents/private knowledge | Teaching the model a new *style*, format, or specialized behavior, not just new facts |
| **Reduces hallucination?** | Meaningfully, by grounding answers in real retrieved text | Not inherently — a fine-tuned model can still hallucinate |

For "answer questions using our internal docs/runbooks," RAG is almost always the right tool over fine-tuning — it's cheaper to keep current, and it can show its source, which matters a lot for trusting an answer used during an actual incident.

---

**Why this is genuinely DevOps-relevant, not just an ML topic**

A vector database is, operationally, **just another stateful service** — something that needs to be provisioned, scaled, backed up, and monitored, the same operational lens applied to any database throughout this repo. And the indexing pipeline (chunk → embed → store) is exactly the kind of **scheduled or event-triggered automation** covered in the DevOps interview's automation questions (Q3/Q4, Interview #2) — it needs to re-run whenever source documents change, which is a genuinely ownable DevOps automation problem: a CI job or scheduled task that re-indexes on a doc-repo push, not a one-time setup step.

---

**Real-world example — CloudCart**

CloudCart has an internal on-call assistant built on RAG, indexed over the team's runbooks and past incident postmortems: when an on-call engineer describes a symptom during an active incident, it retrieves the most relevant past incidents/runbook sections and surfaces them alongside the LLM's synthesized answer — with a link back to the actual source document, specifically so the engineer can verify it rather than blindly trust a generated summary during something time-sensitive. The re-indexing pipeline runs as a scheduled job triggered whenever the postmortems/runbooks repo gets a new commit, so a fix documented today is searchable within minutes, without anyone needing to retrain or redeploy anything model-related. Chunk size was tuned down after an early version retrieved whole, overly long incident documents — too much irrelevant surrounding text diluted the actually-relevant section and produced noticeably less useful answers, a direct, practical instance of the token/context-budget concerns from Q2.

---

**Complete thought process — how I approach this in the interview**

```
What problem does RAG solve?
  → Model training data is frozen/cut off, doesn't include private
    or current org-specific information

How it works — two phases:
  Indexing (offline):  docs → chunks → embeddings → vector DB
  Query time (live):   question → embedding → similarity search →
                        top-K chunks → injected into the PROMPT →
                        LLM generates answer grounded in that context

Critical point: the MODEL doesn't change at all — RAG only changes
what gets retrieved and injected at prompt time, not the weights

Volunteer the natural follow-up unprompted:
  RAG vs fine-tuning — RAG is cheap/fast to keep current and
  traceable to a source; fine-tuning changes the model's actual
  behavior/style, is slow/expensive, and isn't inherently about
  reducing hallucination

Ground it in DevOps ownership, not just ML trivia:
  → Vector DB = just another stateful service to operate
  → Indexing pipeline = a real automation problem — scheduled/
    event-triggered re-indexing when source docs change
```

---

**Summary (what to say if time is short):**

*"RAG, Retrieval-Augmented Generation, grounds an LLM's answer in specific external documents retrieved at query time, rather than relying only on what it learned during training — which matters because training data is frozen and doesn't include an organization's private or current information. It works in two phases: offline, source documents get split into chunks, each chunk gets converted into a vector embedding, and those are stored in a vector database; then at query time, the user's question gets embedded the same way, a similarity search pulls back the most relevant chunks, and those get inserted into the prompt alongside the question so the model generates its answer grounded in that retrieved text. The important mechanical point is that the model itself never changes — RAG only controls what it sees at the moment of answering, unlike fine-tuning, which actually updates the model's weights, is slower and more expensive, and doesn't inherently reduce hallucination the way grounding in real retrieved documents does. I'd also frame this as genuinely DevOps-relevant, not just an ML topic — a vector database is operationally just another stateful service to provision and monitor, and the indexing pipeline that keeps it current is a real automation problem, ideally triggered whenever the source documents change rather than run manually. We actually have an internal on-call assistant built this way at CloudCart, indexed over runbooks and past incident postmortems, and tuning the chunk size down after an early version retrieved too much irrelevant surrounding text was a direct, practical lesson in exactly the token and context-budget concerns from the previous question."*

---

#### Q4. What is MCP protocol? What is Model Context Protocol and why is it used?

**Answer:**

MCP — **Model Context Protocol** — is an open, standardized protocol that defines how an AI application (an LLM client, like Claude Code, or any other MCP-compatible tool) connects to external tools, data sources, and systems. The problem it solves is fundamentally an integration-scaling problem, not a model-capability problem: without a shared standard, every AI application that wants to talk to every external system — a database, GitHub, an internal deployment API, Slack — needs its own **bespoke, one-off integration** for each one. With **N** AI applications and **M** external systems, that's an **N × M** integration-code problem. MCP turns it into **N + M** — a tool/system implements the MCP server side **once**, and it becomes usable by **any** MCP-compliant client without either side writing custom glue code for the other.

---

**The common analogy, and why it's actually accurate**

MCP is often described as **"USB-C for AI applications"** — before USB-C, every device needed its own proprietary connector and cable; standardizing the physical/electrical interface meant any USB-C device works with any USB-C port, regardless of who made either end. MCP does the same thing for the *software* interface between an AI client and an external capability: implement the protocol once on each side, and any compliant pairing just works.

---

**The architecture — client, server, and what a server actually exposes**

```
AI application (MCP CLIENT)  ←→  MCP SERVER  ←→  the actual system
  e.g. Claude Code                e.g. a GitHub          e.g. GitHub's
                                   MCP server              real API
```

An **MCP server** wraps access to one specific system and exposes it through the protocol's standard primitives:
- **Tools** — functions the AI client can actually call (e.g., "create a Jira ticket," "run this read-only database query," "list open PRs") — conceptually similar to function calling, but standardized at the protocol level instead of being a bespoke integration per app.
- **Resources** — data the client can read and pull in as context (a file, a set of database rows, a log excerpt).
- **Prompts** — reusable prompt templates a server can expose for common tasks against that system.

The **MCP client** (the AI application) discovers what tools/resources a connected server offers, and can call them during a conversation — the model decides *when* to use a tool based on the user's request, and the protocol standardizes *how* that call and its result are structured, regardless of which specific server is on the other end.

---

**A genuinely concrete, directly verifiable example — this very session**

Rather than describe this abstractly, I can point to something real: in the Claude Code session I'm using to build this repository, there are tools available with an `mcp__` prefix — Gmail, Google Calendar, Google Drive — which are exactly MCP servers connected into this session, giving the AI client a standardized way to read/act on those systems instead of each capability being hand-built into the AI tool itself. That's MCP in actual, current use, not a hypothetical description.

---

**Why it's used — the actual value proposition**

| Benefit | What it actually means |
|---|---|
| **Standardization** | Solves the N×M integration explosion — implement the protocol once per tool/system, reusable by any compliant AI client |
| **Composability/reuse** | An MCP server built once for an internal system can be used across multiple AI clients and teams, without re-writing integration code for each |
| **A controlled, auditable boundary** | The server — not the raw underlying system — decides exactly what's exposed (e.g., read-only query access only), giving a scoped, inspectable interface instead of the model having unconstrained access |
| **Local or remote** | A server can run locally (e.g., over stdio, for something like local filesystem access) or remotely (over HTTP, for a hosted integration) — flexible depending on where the data/system actually lives |

---

**Why this is genuinely a DevOps/platform concern, not just an AI topic**

Building or operating an MCP server is, operationally, **standing up and securing a privileged service** — the same discipline as any other internal API a DevOps/platform team owns. The concern that matters most: an MCP server that can only **read** something is a very different risk profile from one that can **write/execute** actions (deploy something, delete a resource, send a message on someone's behalf) — this connects directly to the "AI-generated changes need the same review discipline" principle from Q1, just one layer earlier: **scoping what an MCP server is even capable of doing** is the actual security control, the same least-privilege principle threaded through every IAM/RBAC answer in this repo. A well-designed internal MCP server is deliberately narrow — specific, named tools with tightly scoped permissions and full audit logging of every call — not a general-purpose backdoor into a production system handed to a model.

---

**Real-world example — CloudCart**

CloudCart's platform team built an internal MCP server wrapping the runbook/postmortem knowledge base from Q3's RAG example, plus exactly one write capability: creating a Jira ticket — deliberately chosen because it's a low-risk, easily-reversible action, unlike, say, exposing a "restart this production service" tool without much more careful scoping and audit logging first. Any AI client used across the org — Claude Code, an internal chat assistant — can connect to that one server and get the same capability, instead of every team writing its own bespoke integration against the runbook system and the ticketing API separately. Every call the server receives is logged with the identity of the calling session, the same audit discipline CloudCart applies to any other internal service account with write access to a real system.

---

**Complete thought process — how I approach this in the interview**

```
What problem does MCP solve?
  → Without a standard, every AI app needs a bespoke integration
    per external tool/system — N apps × M systems worth of glue code
  → MCP: implement the protocol once per side → N + M instead —
    "USB-C for AI applications"

Architecture:
  MCP CLIENT (the AI app) ←→ MCP SERVER (wraps one system) ←→
  the real system underneath
  → Server exposes: Tools (callable functions), Resources (readable
    data), Prompts (reusable templates)

Concrete, verifiable example: THIS session's Gmail/Calendar/Drive
mcp__ tools are literally MCP servers connected right now — not a
hypothetical

Why it matters, DevOps lens specifically:
  → Standardization/reuse — real integration-cost savings, not just
    a nice abstraction
  → The REAL security lens: scoping what a server can DO is the
    actual control — read-only vs write/execute is a completely
    different risk profile, same least-privilege principle as every
    IAM answer in this repo, just applied to an AI tool boundary
```

---

**Summary (what to say if time is short):**

*"MCP, Model Context Protocol, is an open standard for how an AI application connects to external tools and data sources — the problem it solves is that without a shared protocol, every AI app needs a custom, bespoke integration for every external system, which is an N-times-M integration problem. MCP turns that into N-plus-M: a tool or system implements the MCP server side once, and any MCP-compliant AI client can use it without custom glue code on either end — it's often described as USB-C for AI applications for exactly that reason. An MCP server exposes tools the AI can call, resources it can read as context, and reusable prompt templates, and the client discovers and calls those during a conversation. I can point to a concrete, current example rather than just describe it abstractly — this very Claude Code session has Gmail, Calendar, and Drive available as MCP-connected tools, which is MCP in actual use, not a hypothetical. The part I'd emphasize as the real DevOps-relevant angle is security: building or operating an MCP server is standing up a privileged service, and the real control is scoping exactly what that server is capable of doing — read-only access is a completely different risk profile than a tool that can write or execute actions, which is the same least-privilege principle behind every IAM policy in this repo, just applied one layer earlier, to what an AI tool is even allowed to touch in the first place."*

---

<!-- Add more scenario questions as Scenario 1, Scenario 2... -->

---

<!--
To add a new interview, copy the block below and paste it at the bottom:

## Interview #2

**Company:**
**Date:**
**Role Applied For:**
**Round:**
**Interviewer Level:**

### Questions Asked

#### Q1.

**Answer:**

---
-->
