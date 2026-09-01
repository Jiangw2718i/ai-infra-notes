# Conventions

Notes for whoever adds to this repo (including future me and the agents that help).

## Layout

```
<topic>/<slug>.md        English note      ← default, canonical path
<topic>/<slug>.zh.md     Chinese write-up  ← optional, same slug
```

One directory per topic, not per machine: `dgx-spark/`, later `mac/`, `models/`. A note that
spans hardware goes in the directory of whatever a searcher would name.

The slug describes **the problem**, not the component: `dcgm-skips-on-gb10.md`, not `dcgm.md`.
Someone should be able to guess from the filename whether it is their problem.

English is the default path. Chinese gets the same slug plus `.zh`, so the pair sorts adjacent
in a directory listing. Not every note needs both languages, and neither one waits for the other.

## The two languages are not translations

The Chinese version is the long-form original: how the problem was actually chased, including the
wrong turns. The English version is the extract — Symptom / Cause / Fix, no narrative.

Do not "translate" one into the other. They have different jobs. If the Chinese one is three times
longer, that is correct.

## What a note has to contain

- **Error text verbatim.** That string is the entry point — it is how people arrive. Never
  paraphrase it, never fix its typos, never truncate it.
- **Raw terminal output stays raw**, inside a fenced block. Never reformat command output into a
  markdown table.
- **Source references pinned to a tag and SHA**, e.g. `v4.6.1` (SHA `64df9f89`), with `file:line`.
  A line number without a pinned revision is worthless six months later.
- **Version and hardware context** near the top, so a reader can tell in one line whether this
  applies to them.

## What could not be settled, says so

If a cause was never nailed down, or a fix works without a known reason, write that plainly in the
note. Do not round it up to a confident explanation.

This is the part that is worth the most and it is also the part an LLM will quietly smooth away —
its instinct is to produce an explanation because an explanation is expected there. Everything in
this repo was run on real hardware; a claim that was inferred rather than measured has to say which
one it is, or the whole repo is worth less.

If a later measurement overturns a note, correct the note and keep a line saying what it used to
say and what disproved it. `dgx-spark/` has already had one finding die this way.
