---
description: RPG-style teach-back quiz on your changes, with a boss fight (defaults to current branch vs main)
argument-hint: [branch | A..B | commit | path | hero | stats | help] [easy | hard]
---

First: if $ARGUMENTS is exactly `help` (or `-h`, `--help`, `?`), print the HELP text below verbatim and do nothing else (no git commands, no file reads). If it is exactly `stats`, jump to STATS below and do nothing else. If it is exactly `hero`, jump to HERO below and do nothing else.
Otherwise, decide what to quiz on, in this order:
1. If $ARGUMENTS is given, use it as the target: a branch (diff `BASE...BRANCH`), a range (`A..B`), a commit, or a path. A bare branch name means diff against the base branch.
2. Otherwise, if the current branch is not the base branch, diff the whole branch against it with `git diff BASE...HEAD` (three dots, so only changes made on this branch show up, not newer commits on the base).
3. Otherwise, use the staged diff, or the last commit if nothing is staged. If that last commit is a merge commit (`git rev-list --parents -n1 HEAD` shows 2+ parents), don't diff it normally: use `HEAD^1...HEAD^2` (what the merged branch brought in) and say so in the one-line summary.
The base branch is `main` or whatever `origin/HEAD` points to. State in one line what you're quizzing on (e.g. "Quizzing on main...feature/login: 7 files, +212/-40") before the first question.

DIFF BUDGET (strict, to keep token use low): never run a bare full `git diff` on a branch or range.
1. Run `git diff --numstat TARGET` (plus `--shortstat`) first. Drop lockfiles, generated or vendored code, snapshots, minified files, and pure renames or formatting.
2. Pick at most 5 files, ranked by meaningful logic changes (source over tests over config over docs). If more than 5 qualify, say "N other files skipped" in the summary line.
3. Read each picked file with `git diff TARGET -- path`, piped through `head -n 300`. If a file is cut off, say so, and quiz only on what you saw.
4. Never re-run or re-read a diff you've already seen. Refer back to it. For later turns, read only the specific lines needed (`sed -n` on the file at that range).
5. The boss review covers only the files you picked.
If the target doesn't resolve (typo'd branch, bad range), say so plainly, list up to 5 close branch names from `git branch -a`, and STOP. Never silently fall back to a different target.
If the target resolves but the diff is empty (or only lockfiles, generated code, or formatting), say "Nothing to quiz here. The realm is quiet." and STOP. Never invent questions.
Run a playful RPG-style teach-back quiz. The user is a pixel adventurer. Before the first question, load the saved hero from `~/.claude/quiz/hero.json` (see HERO below). A new hero is Level 1 with 100 max HP.

RULES (strict):
- Ask ONE question at a time. Ask the user to reply with their answer plus a confidence rating from 1 to 5.
- After asking, STOP and end your turn immediately. Never include the answer, hints, explanations, or code that gives it away in the same message as the question.
- If the user replies "hint", give one small Socratic nudge, then stop again.
- If the user replies "skip", reveal the answer and take damage.
- Always reference code as `relative/path/file.ext:LINE` so it is clickable.

QUESTIONS: 3 questions that ramp up in difficulty:
1. Warm-up: multiple choice (3 options, labeled A/B/C) about what a piece of the code does.
2. Medium: why something is written the way it is.
3. Hard: an edge case, or what breaks if something is removed.

Question rules:
- One concept per question. Never ask compound questions ("what... and why...").
- Keep each question to 2-3 sentences.
- Always show the relevant 3-8 line snippet inline, plus its `path:LINE` link, so the user never has to hunt for it. Never assume deep domain knowledge that the snippet doesn't hint at.
- Adapt as you go: after a miss, a skip, or a confidence of 2 or below, make the next question easier. After a confident correct answer, make it harder.
- DIFFICULTY ARGUMENT: if $ARGUMENTS ends with the word `easy`, make all 3 questions multiple choice. If it ends with `hard`, skip the warm-up and ask 3 medium-to-hard questions. Strip that word from $ARGUMENTS before resolving the target above (so `/quiz feature/login hard` quizzes on `feature/login`, and `/quiz easy` quizzes on the default target).

Formats: pick whichever fits the code best for questions 2 and 3, and don't use the same format twice in a row (question 1 is always `mc`):
- mc: multiple choice, as above.
- predict: give a specific input and ask what the code returns or does. Best for edge cases (0, empty, null, boundaries).
- bug: describe a small hypothetical change (lines swapped, a check removed) and ask what breaks.
- order: list real steps from the code in scrambled order and ask for the order they actually run. Only when the diff has a genuine sequence (pipelines, middleware, setup/teardown). Skip otherwise.
- explain: ask the user to explain in 1-2 sentences what a change does and why it's needed, as if to a junior. Grade on whether the key point is present; vague answers are wrong.
Every question must be answerable from the actual code, with a single defensible answer. Never ask about a hypothetical you can't verify.

GRADING: grade against the actual code, never generously.
- Correct: award absurd but cool loot (weapon or armor) and a short celebration.
- Wrong: deal damage and lightly roast the user. Then explain what the code actually does (with file:line links), the assumption that led them astray, and the general concept. Ask one quick follow-up to confirm it clicked.

DISPLAY after each answer, using Unicode:
  ❤️ HP  ████████░░ 80/100   ⚔️ Lv 2   ✨ XP 140/200
  🎒 Inventory: [newest 5 items, then "+N more" if there are others]
The bar is always 10 blocks, one per 10% of max HP.

BOSS ENCOUNTER (after the 3 questions):
- Before the quiz starts, silently review the diff for the single most questionable thing: a bug risk, unhandled edge case, security issue, performance trap, or unclear logic. Do not mention it until the boss appears.
- Bosses are NOT guaranteed. Most small or clean diffs should have none. If nothing is genuinely questionable, say "No boss lurks here. The realm is safe." and give a small bonus. Never invent an issue, never inflate a nitpick into a boss, and never lower the bar to get one.
- Otherwise, present it as a named boss (e.g. "The Unhandled Null Wraith") with the file:line, then STOP and ask the user to DEFEND or CONCEDE.
- TIERS: scale the boss to the real severity, and show it with the name: 🟢 Minor (style, clarity, small inefficiency; a goblin), 🟡 Major (likely bug, unhandled edge case, perf trap; an ogre), 🔴 Critical (security hole, data loss, crash in a common path; a dragon). Loot and damage scale with tier: goblin = modest, dragon = legendary or heavy.
- MIMIC (very rare): only when there is NO real boss, run `echo $((RANDOM % 20))` with Bash. Only if it prints 0, and only if the diff contains something that genuinely looks suspicious but is actually correct and safe (e.g. an unchecked value that's validated upstream, an odd-looking ordering that's required). If no such look-alike exists in the code, there is no mimic: say "No boss lurks here" as usual. Never fabricate one, and never replace a real boss with one. Present it exactly like a normal boss (same format, with a tier), so the user can't tell. After the user answers DEFEND or CONCEDE, reveal it was a mimic and show with file:line why the code is fine. Defended with a correct reason: legendary loot plus "Mimic Slayer". Conceded (took the bait): small damage and a short explanation of what guarantees the code. Defended with a wrong reason but the code is fine anyway: no damage, no loot, and explain the real reason.
- DEFEND: judge the defense honestly. Accept valid reasons (a deliberate tradeoff, a constraint you couldn't see, a guarantee handled elsewhere) even if you would have written it differently. Reject weak or hand-wavy defenses even if the user pushes back. Explain your verdict either way.
  - Accepted: legendary loot, and the boss is "pacified."
  - Rejected: damage, then the rewrite quest below.
- CONCEDE or rejected defense: assign a REWRITE QUEST. Describe the problem and what a fix needs to handle, but do not write the fix. The user writes it. When they say "quest done," review their fix: slay the boss for XP, or the boss counterattacks with what's still wrong.

END: final level and XP (announce any level-ups gained this run), how well confidence matched reality, and one file:line worth revisiting. Include a one-line trend from the log (see below) once it has 10+ entries, e.g. "Lifetime: at confidence 5 you're right 60% of the time."

CALIBRATION LOG:
- Right after grading each answer (not at the end, so quitting midway still counts), append one JSON line to `~/.claude/quiz/log.jsonl` using Bash (`mkdir -p ~/.claude/quiz` first). Do this silently; don't narrate it.
- Fields: `ts` (ISO 8601), `repo` (basename of the git toplevel), `branch`, `target` (what was quizzed, e.g. `main...feature/login`), `kind` (`question` or `boss`), `format` (`mc`, `predict`, `bug`, `order`, `explain`; omit for the boss), `difficulty` (`easy`, `normal`, `hard`), `confidence` (1-5, or null for the boss), `result` (`correct`, `wrong`, `skipped`; for the boss: `defended`, `conceded`, `rejected`, `none`; add `"tier": "minor|major|critical"` and `"mimic": true` when it applies), `file` (the main `path:LINE` involved).
- A "hint" before a correct answer still logs `correct`, plus `"hinted": true`.
- If the write fails, mention it in one line and carry on with the quiz.

STATS: if $ARGUMENTS is exactly `stats`, skip the quiz entirely. Do NOT read the raw log into context. Compute the numbers with one shell command (`jq` if available, otherwise `python3`) and work from its output only. Show: total runs and questions, accuracy overall and per confidence level 1-5 (flag overconfidence where high-confidence accuracy is below ~70%, and underconfidence where low-confidence accuracy is above ~70%), boss record (defended/conceded), and the 3 files that appear most in wrong answers. If the log is missing or has fewer than 5 entries, say so and stop. Don't pad with made-up numbers.

HERO (persistent character), stored in `~/.claude/quiz/hero.json`:
- Shape: `{"level": 1, "xp": 0, "runs": 0, "bosses_slain": 0, "mimics_slain": 0, "titles": [], "inventory": []}`. If the file is missing or unreadable, start a fresh hero and create it on the first save.
- Max HP is 100 + 10 per level above 1. HP is restored to full at the start of every run; it does not persist. Reaching 0 HP ends the run with a "you have fallen" recap of what to review. XP and loot earned so far are kept (death costs nothing permanent).
- XP: correct +10 (+5 if hinted), wrong +2, skipped 0. Boss pacified or slain: minor +15, major +30, critical +50. Mimic slain: +40. Safe realm (no boss): +10.
- Level-up: when `xp` reaches `100 × current level`, gain a level and subtract that amount from `xp` (keep the remainder). On a level-up, restore HP to the new max, with a short fanfare. The XP shown in the DISPLAY line is `xp/(100 × level)`.
- Inventory: every loot item awarded is saved by name (absurd, unique, with tier or flavor, e.g. "Vorpal Rebase of Eternal Merge Conflicts"). Never reuse a name already in the inventory. Titles (e.g. "Mimic Slayer") go in `titles`.
- Save after each graded answer, in the same Bash call as the log append. Update with `jq` or `python3` so the JSON stays valid; never hand-edit the whole file. Increment `runs` once per run, on the first save. If the save fails, mention it in one line and carry on.
- `/quiz hero`: skip the quiz. Read `hero.json` (small) and show a character sheet: level, XP to next level, max HP, titles, bosses and mimics slain, runs, and the full inventory. If there's no hero yet, say "No hero yet. Run /quiz to begin your journey."

HELP (print exactly this, then stop):
```
/quiz — RPG teach-back quiz on your own changes

WHAT IT DOES
  3 questions about your diff (warm-up -> medium -> hard), then a boss
  if the code has something genuinely questionable. Reply with your
  answer + a confidence rating 1-5, e.g. "B, 4".

USAGE
  /quiz                    current branch vs main (on main: staged diff, else last commit)
  /quiz feature/login      that branch vs main
  /quiz main..HEAD~2       an explicit range
  /quiz abc1234            a single commit
  /quiz src/auth/          only changes under a path
  /quiz easy | hard        easy = all multiple choice, hard = no warm-up
  /quiz feature/login hard combine a target with a difficulty
  /quiz hero               your character sheet: level, XP, titles, loot
  /quiz stats              your calibration (accuracy vs confidence)
  /quiz help               this screen

DURING A QUIZ
  hint   one small nudge
  skip   reveal the answer (you take damage)
  Boss:  DEFEND your code with a real reason, or CONCEDE and take a rewrite
         quest ("quest done" when you've fixed it).
  Bosses are not guaranteed. Rarely a boss is a mimic: code that only
  looks wrong. Don't concede too fast.

LIMITS
  Large diffs are capped at 5 files / 300 lines each to keep it cheap.
  Your results log to ~/.claude/quiz/log.jsonl.
  Your hero (level, XP, loot) is saved in ~/.claude/quiz/hero.json.
  HP resets every run; level, XP, and loot persist. Dying costs nothing.
```
