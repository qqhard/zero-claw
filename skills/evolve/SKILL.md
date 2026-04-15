---
name: evolve
description: "Daily self-evolution: add skills/SOUL when patterns emerge, forget what no longer earns its place (both inside skills and in memory). Triggered by heartbeat's last-of-day run, or manually via 'evolve' / 'self-review'."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---

# Evolve

Autonomous daily self-compression. The bot evolves its own capabilities.

## Philosophy: two forces

- **生 (add) — fast, signal-triggered.** Create/modify skills, SOUL.md, or memory when strong patterns emerge. No signal → no change. No budget.
- **忘 (forget) — slow, time-triggered.** Every run, remove what no longer earns its place. Same principle applies inside a skill (cut redundant sections), inside SOUL (drop phrases that stopped resonating), and across memory (prune superseded entries). Only the budgets differ.

The asymmetry — fast birth, slow forgetting — creates a natural filter. Useful additions survive the daily compression pressure; weak additions fade.

"磨" (grinding down a skill) and "忘" (pruning memory) are the same force on different objects. Don't split them in your head.

## Scope

**Allowed to touch:**

- ✅ `.claude/skills/<name>/` — **only if** `<name>` is listed in `.claude/skills/.self-skills` (plain-text registry, one name per line).
- ✅ `SOUL.md`
- ✅ `memory/*`

**Forbidden:**

- ❌ `USER.md`, `IDENTITY.md`, `CLAUDE.md`
- ❌ `journal/*` — journals are raw facts, never rewrite history
- ❌ Any skill **not** in `.self-skills` (plugin-provided skills are third-party mature components — never modify)

If `.claude/skills/.self-skills` does not exist, create it as an empty file. Treat absence as "no self-skills yet".

## Inputs

Read these before deciding anything:

1. Today's journal: `journal/$(date +%Y-%m-%d).md`
2. Last 7 days of journals: `journal/*.md` (recent)
3. Recent commits: `git log --since=7.days.ago --oneline`
4. Recent self-evolution: `git log --grep='^evolve(' --since=30.days.ago`
5. Reverted evolve commits: `git log --grep='^Revert.*evolve(' --since=30.days.ago` — for each, read the reverted diff and note the file + section to avoid.
6. Current `memory/` state: `ls memory/*.md` and read `memory/MEMORY.md`
7. Current self-skills: `cat .claude/skills/.self-skills` (may be empty / missing)

## Phase A — 生 (conditional, no budget)

The bot has been doing the same thing over and over, but has no skill for it, or getting the same correction without it sticking. Name the pattern. Next time it recurs, the bot has a handle for it.

**When to act**: a request pattern appears in journals ≥3 times in the last 7 days with no matching skill, OR the user corrects the same behavior ≥2 times.

**What to write**:

- **A new self-skill**: name it for the pattern; draft a SKILL.md that is *minimal and specific* — solving exactly the cases you saw, not the cases you imagine. The forgetting force will compress whatever survives daily use; better to start thin than pad for "future-proofing". Append the name to `.claude/skills/.self-skills`.
- **A SOUL.md edit**: record what the user actually said or meant, in their own register. Small patch, usually one line under `## Notes from the User`.
- **A memory entry**: capture anything worth remembering that doesn't belong in a skill or SOUL. Short, plain, one file per idea.

**What not to do**:

- Don't invent patterns that only happened once.
- Don't pre-generalize a new skill for hypothetical cases.
- Don't touch anything outside the scope section above.

If in doubt, skip this phase — there's always tomorrow.

## Phase B — 忘 (always, with per-object budgets)

Things grow heavier than they need to be. A skill's first draft guessed at edge cases; the second pass padded guardrails; by week three there's paragraphs nobody reads. Memory entries get superseded or quietly stop mattering. Daily pressure keeps everything honest — only what earns its place survives.

**One question for any candidate**: *"if I remove this, does the bot's capability or knowledge still stand?"* If yes → cut.

This force operates on three kinds of objects, each with its own safety budget. Every run, apply it to each kind once:

### Inside a self-skill (budget: ≤20 lines, one cut)

Pick one self-skill (round-robin over `.self-skills`; prefer the one least recently touched by an evolve commit — `git log --grep='forget:.*<name>' --since=7.days.ago`). If `.self-skills` is empty, skip.

Walk the SKILL.md and find **one** thing whose removal wouldn't break the frontmatter `description` promise. Common candidates: things said twice, guardrails against failures that can no longer happen (use `git blame` to see why they were added), examples that went from illustrative to archival, `allowed-tools` entries not referenced in the body.

Cut it. Diff ≤20 lines.

**Retirement is the limit case**: if a self-skill has been forgotten down to <15 lines of real content and nothing unique remains — delete the folder and drop the name from `.self-skills`. Don't mourn it.

### Inside SOUL.md (opportunistic, no fixed budget)

If a phrase in SOUL.md no longer matches how the bot actually behaves, or it duplicates something said better elsewhere in the file, rewrite or remove it. Only touch SOUL when the mismatch is clear — don't edit for style.

### Across memory (budget: `min(2 files, 5%)`)

Prune redundant `memory/` entries. Always conservative — the point isn't to shrink fast, it's to shrink *steadily* without the user noticing anything important vanished.

What counts as safe to forget (roughly in this order):

1. A newer entry already contradicts/replaces this one.
2. The fact has been promoted into SOUL.md or a skill — it now lives in a more structured form, the raw note is redundant.
3. Old (>90 days), unreferenced by any recent journal, and not load-bearing for anything you can trace.

What **not** to forget:

- Anything the user explicitly said "remember this" about, unless it's been superseded.
- The only copy of a fact (if it's not elsewhere, pruning loses it).
- User profile data — that lives in USER.md, not memory/, but if it's drifted into memory/ by accident, move it, don't delete it.

After pruning, rewrite `memory/MEMORY.md` so the index doesn't point at ghosts.

## Revert learning (stateless)

Before writing anything in any phase:

```bash
git log --grep='^Revert.*evolve(' --since=30.days.ago --format='%H'
```

For each revert commit hash, run `git show <hash>` to see what was undone, then skip any proposed change that would repeat that modification (same file + same region). Source of truth is git history — no separate state file.

## Commit protocol

If any phase produced a change, stage and commit everything in a single commit with this message:

```
evolve(YYYY-MM-DD): <one-line summary>

add:    <what was added, or "none">    (evidence: <journal refs or commit hashes>)
forget: <what was removed, or "none">  (reason: <why it no longer earns its place>)
```

If both phases produced zero changes → **do not commit**. Silent no-op is the correct outcome on a quiet day.

## Safety invariants

- Never run `git push`. The user's local git is the audit trail.
- Never touch forbidden paths (see Scope).
- Never exceed any per-object budget inside Phase B.
- If uncertain whether a proposed change is safe, skip it — there's always tomorrow.
