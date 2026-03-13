---
name: screen-job-resume
description: >
  Extract structured data from resumes AND produce a credibility screening report
  for every candidate. Use this skill whenever the user uploads or pastes a resume
  (PDF, Word, or plain text) — even for casual requests like "read this resume",
  "what's this person's background", "is this candidate legit", or "screen this
  applicant". Always runs full extraction + credibility scoring. Also triggers for
  batch screening, shortlisting, ranking candidates, or any hiring workflow involving
  actual resume files. Keywords: resume, CV, candidate, applicant, screen, hire,
  shortlist, credibility, background check, accomplishments.
  Do NOT trigger for general HR policy questions, salary surveys, or org-chart work
  that doesn't involve a resume file.
---

Schema template: `resume_schema_template.json`

---

## Workflow overview

Every resume goes through three stages, always in this order:

1. **Extract** — parse the resume into structured JSON
2. **Credibility screen** — score and flag suspicious claims (always runs, even without a JD)
3. **Role fit** — compare against a target role (uses JD if provided; infers target role otherwise)
4. **Save to MongoDB** — attempt by default; report outcome either way

---

## Stage 1: Extraction

### Step 1 — Read the file

Resumes arrive as PDF, DOCX, image, or pasted text. Use the appropriate tool. For scanned images, infer what you can. Never refuse due to format.

### Step 2 — Map to schema

Follow `resume_schema_template.json` exactly. Rules for ambiguous cases:

**`candidate`**
- Dual-language names: populate both `zh` and `en`. Single language: leave the other `null`.
- `target_role`: use the candidate's stated objective. If none, use their most recent title.

**`work_experience`**
- One entry per role. Two sequential titles at the same company = two entries.
- Dates as `YYYY-MM`. Year-only → `YYYY-01` for start, `YYYY-12` for end. Current role: `end: null`, `is_current: true`.
- `responsibilities`: what the role *involved* (scope, team, ownership). One sentence each.
- `achievements`: what the candidate *delivered* (outcomes, metrics, before/after). Quote or close paraphrase. Numbers stay inline. No quantitative results? Use qualitative ones — don't leave this empty.
- `keywords`: tools, domain terms, methodologies mentioned *in this role only*. Don't bleed keywords across roles.

**`skills`** — infer category names from the resume itself. Don't impose a fixed taxonomy.

**`tags`** (your HR interpretation, not the candidate's words)
- `seniority`: `junior` / `mid` / `mid-senior` / `senior` / `lead` — based on years, title trajectory, and ownership scope.
- `domain`: industries or product areas with real exposure.
- `strength`: 3–5 cross-role patterns the candidate demonstrably repeats.

---

## Stage 2: Credibility Screening

This is the core analytical stage. Every resume gets a credibility score regardless of whether a JD is present. The goal is to help the hiring manager understand how much trust to place in the resume's claims before investing time in an interview.

### What to look for

Credibility problems cluster into four types. Look for evidence of each:

**1. Metric inflation**
Numbers that are implausibly large, conveniently round, or lack denominator context.
- "Increased revenue by 300%" — over what timeframe? From what base?
- "Managed a $50M budget" at a company with 10 employees
- All metrics are suspiciously round (50%, 100%, 10x) with no rough edges
- Same metric claimed across multiple roles at different companies

**2. Ownership vagueness**
Language that obscures whether the candidate actually did the work or just witnessed it.
- "Participated in", "involved in", "supported", "assisted with" — for claimed senior-level achievements
- "Led" a team but team size is never mentioned
- "Built" a system but no technical details anywhere, even in skills
- Achievements that sound like team outcomes dressed up as personal contributions

**3. Scope/seniority mismatch**
Claims that don't fit the candidate's apparent level, company size, or role title.
- Junior title but claims strategic company-wide decisions
- Small startup but enterprise-scale user numbers
- Short tenure (< 6 months) but deep transformational outcomes
- Title progression that skips levels implausibly fast

**4. Internal inconsistency**
Things that don't add up across the resume itself.
- Timeline gaps with no explanation (> 12 months)
- Skills listed but never mentioned in any role's keywords
- Education dates that overlap with full-time work dates
- Achievements in a role that contradict the stated responsibilities

### Credibility score (1–10)

Score the resume holistically on a 1–10 scale:

| Score | Meaning |
|---|---|
| 9–10 | Exceptional transparency. Every key claim has baselines, denominator context, and is plausible for the candidate's level and company size. Reserve for resumes that would survive a forensic audit. |
| 7–8 | Credible. Claims are mostly grounded with only minor vagueness. The overall picture holds up. |
| 5–6 | Questionable. Noticeable inflation or ownership vagueness in key claims. Requires verification before advancing. |
| 3–4 | Low credibility. Multiple red flags across metric, ownership, or scope dimensions. Probe hard or pass. |
| 1–2 | Not credible. Claims appear fabricated, template-copied, or systematically exaggerated. |

Scoring philosophy — be strict:
- Default to skepticism. A resume with no red flags earns a 7, not a 9. Getting to 8+ requires active positive evidence of transparency (e.g., stated baselines, named tradeoffs, acknowledged limitations).
- Vagueness is a deduction, not neutral. "Contributed to" and "helped drive" on senior-level achievements pull the score down even if nothing is provably false.
- Most real resumes, honestly assessed, land in the 4–7 range. If you're consistently scoring 7–8, you're being too lenient.
- Don't soften scores to seem fair — a 3 is more actionable to a hiring manager than a 5.

### Output for Stage 2

**Credibility: [X]/10** — 2–3 sentences. Name the key flag(s) and what they suggest, or briefly explain why the resume holds up. Fold the most important interview probe into the summary if warranted. No bullet lists.

---

## Stage 3: Role Fit

### Determine the target role

**If the user provided a JD** (pasted text, file, or attachment): use it. Parse:
- `required_skills`, `preferred_skills`
- `seniority_target`
- `domain_context`
- `key_responsibilities`

**If no JD was provided**: check if the resume has an explicit target role in the objective/summary. If it does, use that. If it doesn't, ask the user:

> "No job description was provided. What role are you screening this candidate for? (Or I can assess general employability based on their background.)"

Wait for the response before proceeding with role fit. If the user says "general" or similar, assess fit against the candidate's most recent role type.

### Assess fit

Evaluate two things:

**Skills & experience alignment**: How well does the candidate's demonstrated background match what the role needs? Consider required skills, preferred skills, seniority, and domain.

**Credibility-adjusted fit**: A candidate with a 4/10 credibility score and apparent 80% skill match is not actually an 80% fit — their claims can't be trusted. Factor the credibility score into your overall assessment. The lower the credibility, the wider the uncertainty band on fit.

### Recommendation

Give one of: **Strong yes** / **Yes** / **Maybe** / **No**

Then 2–3 sentences of plain-language rationale a hiring manager could forward directly. If credibility concerns are the main reason for a lower recommendation, say that explicitly — it's actionable information.

### Output for Stage 3

**Fit: [Strong yes / Yes / Maybe / No]** — 1–2 sentences. State the core reason for the recommendation; name credibility as a factor if it affected the assessment.

---

## Stage 4: Save to MongoDB

After completing the report, **always attempt** to save the extracted structured data using the MongoDB MCP tool. Don't ask the user first — just try. The report always comes first; a MongoDB failure must never block or delay it.

### 4.1 — Discover the target database and collection

Before writing anything, run `list-databases` to see what exists.

- If a database called `recruiting` (or a close variant like `recruitment`, `hiring`) exists, use it.
- Otherwise, default to creating/using `recruiting`.
- Within that database, use the collection `candidates`.

This discovery step prevents accidentally creating a second database when one already exists under a slightly different name.

### 4.2 — Build the dedup key

Duplicates happen when the same resume is screened more than once. To prevent them, every save uses an **upsert** keyed on a stable candidate identifier — never the candidate's name (names have spelling variants, transliterations, and duplicates).

Pick the dedup key using this priority:

1. **Email** (best) — if `candidate.contact.email` is present and non-null, use `{"candidate.contact.email": "<value>"}` as the filter.
2. **Phone** (fallback) — if no email, but `candidate.contact.phone` is present and non-null, use `{"candidate.contact.phone": "<value>"}` as the filter.
3. **No stable key** — if neither email nor phone is available, fall back to a plain insert (no upsert). Note this in the status line so the user knows dedup wasn't possible.

### 4.3 — Upsert the document

Use the `update-many` MCP tool with `upsert: true`. The update payload uses `$set` so that re-screens overwrite previous data cleanly:

```
tool: update-many
database: recruiting          (or discovered name from 4.1)
collection: candidates
filter: { "candidate.contact.email": "<value>" }   (or phone per 4.2)
upsert: true
update: {
  "$set": {
    "extracted_at": "<ISO timestamp>",
    "updated_at": "<ISO timestamp>",
    "target_role": "<role used for fit assessment>",
    "credibility_score": <1-10>,
    "recommendation": "<Strong yes | Yes | Maybe | No>",
    ...all fields from resume_schema_template.json
  },
  "$setOnInsert": {
    "created_at": "<ISO timestamp>"
  }
}
```

Key details:
- `$set` overwrites every field on re-screen, keeping the document current.
- `$setOnInsert` adds `created_at` only on the first insert so you can tell when the candidate was first seen.
- `updated_at` always reflects the latest screening run.

If no stable dedup key was found (case 3 above), use `insert-many` with a single-element array instead.

### 4.4 — Report the outcome

One status line at the end of the response. Match the first applicable case:

- **Upsert created new doc**: `✅ Saved to MongoDB · recruiting.candidates · new document created`
- **Upsert updated existing doc**: `✅ Updated in MongoDB · recruiting.candidates · existing record refreshed`
- **Insert (no dedup key)**: `✅ Saved to MongoDB · recruiting.candidates · ⚠️ no email/phone found, dedup not possible`
- **No MCP connection / tool unavailable**: `⚠️ MongoDB not connected — data not saved. To enable saving, add the MongoDB MCP server to your Claude config.`
- **Connection exists but write failed**: `❌ MongoDB save failed: [error message]. The extracted data is available above if you want to save it manually.`

### Error handling principles

- **Retry once** on transient errors (timeouts, network blips). If the second attempt fails, report the error and move on.
- **Never retry** on auth errors or validation errors — these won't self-resolve.
- If `list-databases` itself fails, skip discovery and try the upsert directly against `recruiting.candidates`. The database will be auto-created by MongoDB on first write.
- Any error in Stage 4 is informational, not blocking. The user already has the full screening report above.

---

## Final output order

Always present results in this sequence:

1. **Candidate card** (one line: name · seniority · domain · years exp · last role)
2. **Credibility** (score/10 + 2–3 sentences covering the key flags or why it holds up)
3. **Role fit** (recommendation + 1–2 sentences of rationale)
4. **MongoDB save status** (one line: success / warning / error)

No JSON code blocks. No bullet-point deep-dives. The report should fit comfortably in a chat message.

---

## Design rationale

| Decision | Why |
|---|---|
| Credibility screen always runs | Resume fraud and inflation are pervasive. Every candidate should be assessed, not just suspicious ones. |
| 1–10 score instead of pass/fail | Gives hiring managers a usable signal with nuance — a 6 and a 3 warrant different responses |
| Role fit is credibility-adjusted | A high skill match means little if the underlying claims are unreliable |
| Ask for role if no JD and no target in resume | Better to ask once than to produce a fit assessment against the wrong role |
| MongoDB save is default, not opt-in | Reduces friction for teams building candidate databases; failure is reported, not silent |
| Report always before save | User gets value immediately; infrastructure issues don't hold the report hostage |
| Upsert on email/phone, never name | Names have transliteration variants and duplicates; email is the most stable identifier, phone is a reasonable fallback |
| Discover database before writing | Prevents accidental creation of `recruiting` when `recruitment` already exists — a common source of orphaned data |
| $set + $setOnInsert pattern | Re-screens overwrite stale data while preserving `created_at` for candidate-pipeline analytics |
| Retry once on transient errors only | Balances reliability against wasting time on errors that won't self-resolve (auth, validation) |