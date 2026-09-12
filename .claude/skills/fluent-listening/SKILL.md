---
name: fluent-listening
description: Run an interactive listening session built around audio the learner plays themselves. Triggered only when the learner types /fluent-listening. Runs gist questions, detail questions, and dictation with the transcript hidden until after each round, then diagnoses exactly which sounds the learner mis-heard (connected speech, weak forms, minimal pairs) and drills them with shadowing.
allowed-tools: Read, Write, Bash
disable-model-invocation: true
---

# Listening Session

## Overview

Claude cannot play audio. The learner plays it; this skill runs everything around
it — gist checks, detail questions, dictation, mis-hearing diagnosis, shadowing.

The load-bearing mechanic is **dictation**: the learner types what they heard, and
the gap between what they typed and what was actually said reveals precisely which
sounds their ear drops. That diff is impossible to get from reading practice and is
the single highest-signal listening exercise that works in a text interface.

## When to Use

Trigger only when the learner types `/fluent-listening`. Gated with
`disable-model-invocation: true` — a 20-25 min session with DB writes must never
start from an ambiguous prompt.

Requires the learner to have audio available and be able to play it. If they
cannot right now, say so plainly and route to `/fluent-reading` instead. Do not
fake a listening session with a transcript.

## Instructions

### 1. Load context

```bash
python3 "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/read-db.py"
```

Need: `learner_profile` (name, level, languages, motivation),
`mastery_db.skills.listening`, and past `mistakes_db.error_patterns` whose
category is `listening`.

### 2. Opening

```markdown
# 🎧 {target_language} Listening Practice

Hi {name}!

I can't play audio — you press play, I handle everything else.

**Today:** gist → details → dictation → shadowing
**Level:** {CEFR}
**Duration:** 20-25 min

**You'll need:** headphones, and the ability to pause/replay.

**Ready?** Paste a link to what you want to listen to, or type **"pick one"**
and I'll suggest something at your level.
```

### 3. Establish the audio source

Two paths:

**Learner supplies it** — a link, a podcast episode, a meeting recording, a
YouTube video. Ask for the approximate length and whether a transcript or
subtitles exist. If no transcript exists anywhere, dictation still works: the
learner types what they heard and reports back what they later confirm. Say
clearly that diagnosis is weaker without ground truth.

**Skill suggests it** — pull from the Source Bank below, matched to
`learner_profile.learner.motivation` and level. Give 2-3 options with length and
why each fits. Let them choose.

Then fix the working segment: **30-90 seconds**, not the whole episode. Longer
segments produce vague "I didn't catch it" results instead of locatable errors.

```markdown
## 🎯 Today's Segment

**Source:** {name}
**Segment:** {start}-{end} (~{N} seconds)
**Transcript available:** {yes/no}

Play it **once**, at normal speed, without transcript or subtitles.
Don't replay yet. Type **"done"** when finished.
```

### 4. Round 1 — gist (transcript hidden)

After one listen only. Two or three questions, one at a time, about the whole
segment: who is speaking, what about, what tone, what happens.

```markdown
## Round 1: Gist — question 1 of {N}

{question}

a) {option}
b) {option}
c) {option}

**Type a, b, or c.** Guessing is fine — say so if you're guessing.
```

Do not reveal the transcript. Give the answer and a one-line reason, and note
whether they guessed. Guessed-correct is not the same as heard-correct; track it.

### 5. Round 2 — detail (one replay allowed)

```markdown
Now play it again — still no transcript.
Listen for specifics this time. Type **"done"** when finished.
```

Then 2-4 detail questions, one at a time: numbers, names, times, a specific
claim, a specific word. Numbers and dates deserve at least one question every
session — they are where comprehension fails in real meetings.

### 6. Round 3 — dictation (the core)

Pick **2-4 short stretches**, 5-12 words each. Choose stretches that carry
connected speech, weak forms, or vocabulary at the learner's edge — not the
easy parts.

```markdown
## Round 3: Dictation — {i} of {N}

Play **{timestamp}-{timestamp}** — just those few seconds. Replay as often as
you want.

Type **exactly what you hear**, word for word. Guess at anything unclear rather
than leaving it blank — a wrong guess tells me more than a gap does.
```

Wait for each before giving the next. Never show the true text first.

### 7. Diagnose the mis-hearing

This is where the session earns its keep. For each dictation, show the diff and
name the *mechanism*, not just the error.

```markdown
**You heard:** {learner's text}
**Actually:**  {true text}

**What happened:**

| You typed | Actually | Why your ear did that |
|---|---|---|
| {x} | {y} | {mechanism} |
```

Mechanisms to name explicitly:

- **Linking** — final consonant binds to next vowel (`an hour` → "a-nour")
- **Elision** — a sound disappears (`next day` → "nex day")
- **Assimilation** — a sound changes to fit its neighbour (`ten boys` → "tem boys")
- **Weak forms** — function words reduce to schwa (`can` → /kən/, `to` → /tə/,
  `of` → /əv/). The single largest cause of "they spoke too fast."
- **Minimal pairs** — two words separated by one phoneme the learner doesn't
  distinguish yet
- **Word-stress misplacement** — heard the syllables, mapped them to the wrong word
- **Unknown vocabulary** — no amount of replay recovers a word never learned.
  Distinguish this from the others: it's a vocab gap, not an ear gap, and the fix
  is different.

Check the learner's native language in `learner_profile.learner` for known L1
interference. For a **Vietnamese** L1 learner of English, expect and watch for:

| Pattern | What it sounds like |
|---|---|
| Final consonants dropped | `worked` heard as `work`, `beans` as `bean` |
| `-s` / `-ed` endings missed | plurals and past tense vanish |
| Consonant clusters simplified | `texts`, `asked`, `strengths` collapse |
| /θ/ /ð/ → /t/ /d/ | `think`/`tink`, `they`/`day` |
| /l/ vs /n/ final | `fall`/`fawn` |
| Syllable-timed hearing | English stress-timing makes unstressed words vanish |

Name the pattern once and it stops being noise. That is the whole point of
listening practice at this level.

### 8. Shadowing drill

```markdown
## 🗣️ Shadowing

Take the stretch you found hardest: **"{text}"**

1. Play it. Don't speak — just listen 3 times.
2. Play it and speak *along with* the audio, same speed, same rhythm.
3. Repeat until you can keep up without falling behind.

**Stress falls on:** {STRESSED words in caps}
**Linking happens at:** {points}

Type **"done"** when it feels smooth — or **"stuck"** and I'll break it into
smaller pieces.
```

Shadowing trains the ear through the mouth: producing the rhythm makes the
learner hear it. Include it every session, even briefly.

### 9. Session summary

```markdown
## 📊 Listening Session Complete!

**Source:** {name} — {segment}
**Listens:** {N} | **Dictations:** {N}

### Accuracy
- Gist: {x}/{y} {note if any were guessed}
- Detail: {x}/{y}
- Dictation: {percent}% of words exact

### What your ear is dropping
1. **{mechanism}** — {count}x — {one-line example}
2. **{mechanism}** — {count}x — {one-line example}

### Unknown words (vocab gap, not ear gap)
{list — offer to add to spaced repetition}

### Next session
- {targeted drill based on the top mechanism}

**{encouragement}** 🎧
```

### 10. Update databases

Use the `fluent-db-updater` skill:

- `command_used: "/fluent-listening"`, `skills_practiced: ["listening"]`
- `skill_scores.listening: {exercises, correct, time_minutes}` — count each gist
  question, detail question, and dictation stretch as one exercise; a dictation
  counts correct only at ≥90% word accuracy
- `errors[]` — one per mechanism, `category: "listening"`, `subcategory` one of
  `linking` / `elision` / `assimilation` / `weak_forms` / `minimal_pairs` /
  `word_stress` / `numbers_dates` / `gist`. Put the learner's typed text in
  `your_answer` and the true text in `correct_answer`.
- `new_vocabulary[]` — unknown words the learner chose to keep
- `focus_next_session[]` — the top mechanism

Save the transcript, dictation diffs, and diagnosis to
`/results/fluent-listening-session-{NNN}.md`.

## Examples

### Example 1 — dictation diagnosis (English, Vietnamese L1)

> **You heard:** *"I would of ask him about the project deadline"*
> **Actually:**  *"I would've asked him about the project deadline"*
>
> **What happened:**
>
> | You typed | Actually | Why your ear did that |
> |---|---|---|
> | `would of` | `would've` | Weak form — `have` reduces to /əv/, identical to `of`. Everyone hears this wrong; native speakers write it wrong too. |
> | `ask` | `asked` | Final cluster /skt/ simplified. Vietnamese doesn't stack three consonants, so your ear trimmed it. |
>
> Both are rhythm, not vocabulary — you knew every word. That's good news:
> it's an ear problem, and ear problems fix fast with shadowing.

### Example 2 — routing away when audio isn't available

Learner: `/fluent-listening` … "I'm on the train without headphones"

> Then this isn't the session for right now — listening without being able to
> replay properly just builds frustration.
>
> Two options: `/fluent-reading` works fine on a train, or `/fluent-vocab` for a
> 10-minute drill. Come back to listening tonight and we'll do the dictation
> round properly.

## Critical Rules

- **Never reveal the transcript before the learner answers.** Showing it first
  converts the session into reading practice. This is the rule that makes the
  skill worth having.
- **Never fabricate a transcript.** If the learner supplies audio and you cannot
  verify the words, say so, and run dictation as self-check — do not invent what
  was said.
- **Segments of 30-90 seconds.** Whole episodes produce unusable diagnosis.
- **Name the mechanism, not just the error.** "You missed a word" teaches
  nothing; "that's a weak form — `to` became /tə/" teaches permanently.
- **Separate ear gaps from vocab gaps.** Unknown words are a vocabulary problem
  and go to spaced repetition; mis-heard known words are an ear problem and go
  to shadowing. Different fixes.
- **Track guessing.** A guessed-correct gist answer is not comprehension. Ask,
  and record it honestly.
- **One question at a time**, waiting for each answer.
- **Never auto-invoke.** Explicit `/fluent-listening` only.

## Source Bank — English, A1 → B1, work-focused

| Source | Level | Length | Transcript | Why |
|---|---|---|---|---|
| VOA Learning English | A1-A2 | 3-5 min | Yes, full | Read at ~2/3 speed, clear US English. The right starting point below A2. |
| BBC 6 Minute English | A2-B1 | 6 min | Yes, full | Two speakers, natural back-and-forth, everyday and work topics. |
| BBC English at Work | A2-B1 | 5 min | Yes | Office scenarios — meetings, email, phone calls. Matches a work motivation directly. |
| All Ears English | B1+ | 15 min | Partial | Fast, unscripted, real connected speech. Use short segments only. |
| The learner's own meeting recordings | any | — | No | Highest transfer value. Accents and vocabulary they actually face. Use once past A2. |

Rotate sources. A learner who only ever hears one voice learns that voice, not
the language — for meetings especially, deliberately mix accents (US, UK, Indian,
Japanese-accented English) once past A2.
