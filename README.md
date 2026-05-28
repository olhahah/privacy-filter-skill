# privacy-filter-skill

A Claude Code skill that strips personal and sensitive content from a transcript while keeping the work content intact.

It cuts greetings/weather/weekend plans, third-party medical details, comp talk, mental-health specifics, and similar — and when a sentence mixes work and personal context (e.g. "X was delayed because my wife is sick"), it keeps the work fact and drops the personal reason. Every removal goes to a local log file so you can review what was cut.

This is one of two skills used together to ship meeting transcripts to a private GitHub repo. The other is [granola-share-skill](https://github.com/olhahah/granola-share-skill) — it's what most people will invoke directly. This filter runs underneath it.

## What you need first

- Claude Code installed (`https://claude.com/claude-code`)
- This skill installed in `~/.claude/skills/privacy-filter/` (see install below)

## Install

```bash
mkdir -p ~/skills
git clone https://github.com/olhahah/privacy-filter-skill ~/skills/privacy-filter

mkdir -p ~/.claude/skills
ln -s ~/skills/privacy-filter/skill ~/.claude/skills/privacy-filter
```

Three commands. The symlink means `git pull`-ing in `~/skills/privacy-filter` is enough to pick up new versions — no re-install step.

## Update

```bash
cd ~/skills/privacy-filter && git pull
```

That's it. The skill is just a markdown file Claude reads — there's nothing to rebuild.

## Use

You'll usually not invoke this directly — the `granola-share` skill calls it for you as part of pushing transcripts. If you want to run it directly on some text:

> "filter this transcript using the privacy filter"

Paste the transcript, Claude applies the rules, prints the filtered version, and appends a log entry per removal to `_logs/privacy-redactions.md` in your current directory.

## What gets cut

- **Always cut:** greetings/weather/weekend plans, compensation, job-search talk, mental-health specifics, third-party medical (partners, kids, parents), confidentiality cues ("between us"), emotional confessions, personal attacks on named colleagues.
- **Likely cut (logged for review):** mild emotional language tied to work, opinions on named colleagues, caregiving phrases without medical detail, self-deprecating work feelings.
- **Always kept:** the employee's own holiday/sick leave, project impact statements (even when emotionally framed), neutral work friction, mentions of team/role/project/customer/metric.
- **Mixed work + personal:** the filter keeps the work fact and drops the personal reason. "X was delayed because my wife is sick" → "X was delayed."

## What does NOT get blocked

The filter **never blocks**. It makes calls in the moment and lets the transcript through. Safety comes from the log review afterwards — you scan `_logs/privacy-redactions.md` and either approve or revert any individual removal.

## The log

Every removal is appended to `_logs/privacy-redactions.md` in whatever directory you're running from. Entry format:

```
## <date> — <transcript title>

- **Confidence:** high | low
- **Speaker:** <UNKNOWN by default when no speaker labels are available>
- **Reason:** <plain-English reason>
- **Quote:** "<full text removed>"
- **Work fact preserved (if any):** <what was kept inline, or "none">
- **Review:** _<blank — fill in after you read>_
```

Mark `Review:` as `✓ correct` / `✗ should have kept` / `~ partial` when you scan the log. Those decisions inform future refinements to this skill.

## Limitations

- **No speaker labels expected.** This skill works on raw text without speaker attribution. The "background bleed" category (cutting lines from people outside the meeting) only fires when context makes it obviously off-meeting — otherwise it leaves the text in.
- **No glossary lookups.** Lighter than MVP-brain's version: no per-org name list, no person-specific rules. If you need that, run the full MVP-brain pipeline on your own machine after pulling.

## Related

- [granola-share-skill](https://github.com/olhahah/granola-share-skill) — the orchestration skill that uses this filter to push Granola transcripts to a private GitHub repo.

## Questions

Olha — olha.dolinska@healf.com
