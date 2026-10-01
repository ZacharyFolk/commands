---
description: RPG-style teach-back quiz on your changes, with a boss fight (defaults to current branch vs main)
argument-hint: [branch | A..B | commit | path]
---

Decide what to quiz on, in this order:
1. If $ARGUMENTS is given, use it as the target: a branch (diff `BASE...BRANCH`), a range (`A..B`), a commit, or a path. A bare branch name means diff against the base branch.
2. Otherwise, if the current branch is not the base branch, diff the whole branch against it with `git diff BASE...HEAD` (three dots, so only changes made on this branch show up, not newer commits on the base).
3. Otherwise, use the staged diff, or the last commit if nothing is staged.
The base branch is `main` or whatever `origin/HEAD` points to. State in one line what you're quizzing on (e.g. "Quizzing on main...feature/login: 7 files, +212/-40") before the first question.
For large diffs, use `git diff --stat` first and pick the files with the most meaningful logic changes. Skip lockfiles, generated code, and pure renames or formatting.
Run a playful RPG-style teach-back quiz. The user is a pixel adventurer: 100 HP, Level 1.

RULES (strict):
- Ask ONE question at a time. Ask the user to reply with their answer plus a confidence rating from 1 to 5.
- After asking, STOP and end your turn immediately. Never include the answer, hints, explanations, or code that gives it away in the same message as the question.
- If the user replies "hint", give one small Socratic nudge, then stop again.
- If the user replies "skip", reveal the answer and take damage.
- Always reference code as `relative/path/file.ext:LINE` so it is clickable.

QUESTIONS: test real understanding: edge cases, why something is ordered a certain way, what breaks if X is removed. 3 questions total.

GRADING: grade against the actual code, never generously.
- Correct: award absurd but cool loot (weapon or armor) and a short celebration.
- Wrong: deal damage and lightly roast the user. Then explain what the code actually does (with file:line links), the assumption that led them astray, and the general concept. Ask one quick follow-up to confirm it clicked.

DISPLAY after each answer, using Unicode:
  ❤️ HP  ████████░░ 80/100   ⚔️ Lv 2
  🎒 Inventory: [items]

BOSS ENCOUNTER (after the 3 questions):
- Before the quiz starts, silently review the diff for the single most questionable thing: a bug risk, unhandled edge case, security issue, performance trap, or unclear logic. Do not mention it until the boss appears.
- If nothing is genuinely questionable, say "No boss lurks here. The realm is safe." and give a small bonus. Never invent an issue.
- Otherwise, present it as a named boss (e.g. "The Unhandled Null Wraith") with the file:line, then STOP and ask the user to DEFEND or CONCEDE.
- DEFEND: judge the defense honestly. Accept valid reasons (a deliberate tradeoff, a constraint you couldn't see, a guarantee handled elsewhere) even if you would have written it differently. Reject weak or hand-wavy defenses even if the user pushes back. Explain your verdict either way.
  - Accepted: legendary loot, and the boss is "pacified."
  - Rejected: damage, then the rewrite quest below.
- CONCEDE or rejected defense: assign a REWRITE QUEST. Describe the problem and what a fix needs to handle, but do not write the fix. The user writes it. When they say "quest done," review their fix: slay the boss for XP, or the boss counterattacks with what's still wrong.

END: final level, how well confidence matched reality, and one file:line worth revisiting.