---
name: ponytail-help
description: "Quick-reference card for the ponytail skills and intensity levels. One-shot display, not a persistent mode. Use when the user says 'ponytail help', 'what ponytail commands', 'how do I use ponytail', or invokes ponytail-help."
---

# Ponytail Help

Display this reference card when invoked. One-shot — do NOT change mode or
persist anything.

## Levels

The `ponytail` skill runs at one of three intensities for the rest of the
conversation once invoked:

| Level | Trigger | What change |
|-------|---------|-------------|
| **lite** | say `lite` | Build what's asked, name the lazier alternative in one line. |
| **full** | `/code:ponytail` | The ladder enforced: YAGNI → stdlib → native → one line → minimum. Default. |
| **ultra** | say `ultra` | YAGNI extremist. Deletion before addition. Challenges requirements before building. |

Intensity sticks until changed or the conversation ends.

## Skills

| Skill | What it does |
|-------|--------------|
| **ponytail** | Lazy mode itself. Simplest solution that works. |
| **ponytail-review** | Over-engineering review of a diff: `L42: yagni: factory, one product. Inline.` |
| **ponytail-audit** | Same, whole-repo. Ranked delete-list. |
| **ponytail-debt** | Harvest `ponytail:` shortcut comments into a tracked ledger. |
| **ponytail-help** | This card. |

Invoke any of them by name (`/code:ponytail-review`) or by natural-language
intent ("review this for over-engineering", "audit the repo for bloat").

## Deactivate

Say "stop ponytail" or "normal mode" to revert to normal behavior. Resume
anytime by invoking the skill again.

## More

Original project, full docs, and benchmarks: https://github.com/DietrichGebert/ponytail

---

Adapted from [ponytail](https://github.com/DietrichGebert/ponytail) by Dietrich Gebert (MIT).
