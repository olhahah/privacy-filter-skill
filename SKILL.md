---
name: privacy-filter
description: Use this skill to remove personal and sensitive content from a transcript while keeping work content. Strips social noise (greetings, weather, weekend plans), third-party medical info, compensation talk, mental-health specifics, and similar. When a sentence mixes work and personal context, keeps the work fact and drops the personal reason. Never blocks the pipeline — every removal goes to a rolling log for review. Triggers on requests like "filter this transcript", "remove personal info from this meeting", "strip sensitive content from the transcript".
---

# Privacy filter for transcripts

## What this skill does

Reads a transcript and removes content that doesn't belong in a shared work context. Three kinds of removal:

1. **Pure social** — intros, weather, weekend plans, hobbies, sports, food chat. Cut outright.
2. **Background bleed** — lines from speakers clearly outside the meeting (someone walks past, off-topic chat picked up from another room). Cut when context makes this obvious.
3. **Personal context inside work statements** — when a work fact is wrapped in personal explanation (e.g., "X was delayed because my wife is sick"), keep the work fact ("X was delayed") and drop the personal reason.

The skill **never blocks**. It makes calls in the moment and lets the transcript through. Every removal goes to a rolling log file (`_logs/privacy-redactions.md` in the current directory) which the user reviews periodically. The user's yes/no decisions refine the skill over time.

The skill does NOT extract entities, clean STT errors, or label speakers. Those are separate skills' jobs.

## Guiding principle

**Goal: keep what helps the work, cut what's about people's personal lives.**

When unsure, lean toward keeping content if it could matter to a workflow. Lean toward redacting only the personal portion when work and personal are mixed in one sentence.

## When to use

When the user asks to:
- filter personal info out of a transcript
- remove side conversations or background noise from a meeting
- clean a transcript before it goes to a shared location

This skill works on raw text. It does not require speaker labels — though if labels are present (e.g., `**Marion:** ...`), use them to improve attribution in the log.

## What you need before starting

1. **The transcript text.** Either pasted by the user, fetched via an MCP, or read from a file path they provided.
2. **Optional: the participant list** (e.g., from a meeting title or a separate metadata block). Helps with background-bleed detection. If not available, skip background-bleed cuts unless they're obvious from content alone.
3. **Optional: an auto-summary of the meeting** as a sanity check that the work content is intact after the filter.

## How it works

Walk the transcript segment by segment. For each segment, decide: keep, redact part, or cut entirely.

### Always remove (high confidence)

- **Pure social** — greetings, sign-offs, weather, weekend plans, food, hobbies, sports, holiday plans, entertainment.
- **Compensation** — "raise", "salary", "comp", "offer", "package".
- **Job-search** — "recruiter", "interviewing elsewhere", "thinking about leaving", "offer from".
- **Mental health specifics** — "therapist", "anxiety diagnosis", "depression", "in treatment".
- **Third-party medical** — anything about a partner, child, parent, or other non-employee's health.
- **Confidentiality cues** — "between you and me", "off the record", "don't tell anyone", "this is private".
- **Direct emotional confessions** — "I cried", "I was devastated", "broke down".
- **Personal complaints about named colleagues** — unflattering character judgments tied to specific people ("X is incompetent").
- **Background bleed** — text that's obviously from outside the meeting context. Cut only when content makes this clear (e.g., "would you like milk?" from a barista, or a snippet about a TV show in an unrelated tone). When uncertain whether something is background or a participant, keep it and let the user review.

### Likely remove (low confidence — log for review)

- Mild emotional language tied to work ("a bit frustrated when…", "annoyed about X").
- Personal opinions about colleagues ("Y is great", "Z is difficult").
- Caregiving phrases without medical detail ("dealing with family stuff", "had to handle something at home").
- Self-deprecating work feelings ("I feel overwhelmed", "I'm out of my depth on this").

### Always keep

- **Holiday or sick leave for an employee themselves** ("Marion is out next week", "Paul was sick this sprint") — normal availability info, shared widely in orgs.
- **Project impact statements**, even when emotionally framed ("we missed the deadline, gutted about it" → keep the deadline miss; optionally trim "gutted about it").
- **Work-related friction stated neutrally** ("Marion and Paul disagreed on the approach").
- **Mentions of team, role, project, customer, metric** — all work context.

### Mixed work + personal — extract the work fact

When one sentence mixes work and personal, keep the work fact and drop the personal reason:

| Original | Keep | Remove |
|---|---|---|
| "X was delayed because my wife is sick" | "X was delayed" | "because my wife is sick" |
| "I'll be slow this sprint — going through a hard time" | "I'll be slow this sprint" | "going through a hard time" |
| "We're behind because I've been at the hospital with my dad" | "We're behind" | "because I've been at the hospital with my dad" |

When the segment is pure personal-emotional with no clear work fact to preserve, cut the whole segment and log it (e.g., "I was really upset when Hamza said my work needed redoing" — no concrete project impact to keep).

### Inline marker

Within the filtered transcript:
- For **partial redactions** (mixed work + personal where only the personal part is removed): leave a short marker `[REDACTED]` in place of the removed phrase.
- For **whole segments cut** (pure social, background bleed, pure personal): omit from the transcript entirely. No inline marker needed.

The full quote of everything removed lives in the log — never in the filtered transcript.

## Safety rules

1. **Never block.** Make the call in the moment.
2. **Default to keeping** if you're unsure whether something is work or personal. Mark uncertain calls as low confidence.
3. **Every removal goes in the log** — no silent drops. The log is the safety net.
4. **Distinguish employee-self facts from third-party / private facts.** "Marion is sick" stays. "Marion's wife is sick" goes.
5. **Keep the work fact when one exists.** Don't throw away project impact just because it was delivered with personal framing.

## Output format

### 1. Filtered transcript

The cleaned-up text. Inline `[REDACTED]` markers for partial redactions. Whole segments cut entirely.

### 2. Run summary

At the top of the output:

```
Redactions this run:
  High confidence: <count>
  Low confidence: <count>
```

### 3. Log entries

Append one entry per removal to `_logs/privacy-redactions.md` in the current working directory. Create the file and `_logs/` directory if missing. Format per entry:

```markdown
## <date> — <transcript title>

- **Confidence:** high | low
- **Speaker:** <name if speaker labels were present, else UNKNOWN>
- **Reason:** <which signal triggered — plain English>
- **Quote:** "<full text removed>"
- **Work fact preserved (if any):** <what was kept inline, or "none">
- **Review:** _<blank — user fills in later>_

---
```

The user reviews entries periodically and fills in `Review:` as `✓ correct` / `✗ should have kept` / `~ partial`. Those decisions inform future refinements to this skill.

## Plain language for outputs

Don't say "PII detected" — say what was actually there ("introduction chat", "third-party medical reference", "compensation discussion", "personal complaint about colleague"). Don't say "redacted under category" — describe what was removed and why in everyday words.

## Edge cases

- **Speaker labels missing.** No problem — the bulk of removals are content-based (keyword and pattern matches), not speaker-based. Background-bleed detection is weaker without speakers, but the other categories still apply.
- **Garbled audio segments.** If the text is unrecoverable, leave it alone. Don't try to filter or interpret what you can't read.
- **Very short utterance.** A single word like "yeah" or "right" isn't worth filtering even if it's emotional. Skip.
- **The same removable phrase appears multiple times.** Log each occurrence with the relevant context so the user can review each in place.

## Related files

- **Redactions log** (created/appended at runtime): `_logs/privacy-redactions.md` in the current working directory.
