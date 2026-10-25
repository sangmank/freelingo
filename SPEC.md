# SPEC.md — delta spec for the sangmank/freelingo fork

**Status: DRAFT. Nothing here is built. Every section below is a statement of intent, not of
current behavior.**

This file describes only what this fork intends to add or change. It does not restate upstream
behavior. For how FreeLingo works today, read upstream's own specs — they are the source of truth
and this fork does not copy them:

- `specs/architecture.instructions.md` — overall shape
- `specs/architecture-backend.instructions.md`, `specs/architecture-frontend.instructions.md`
- `specs/services.instructions.md`, `specs/speech-services.instructions.md` — LLM/TTS/STT contracts
- `specs/voice-conversation.instructions.md` — the `/ws/conversation` flow
- `specs/multi-language.instructions.md`, `specs/target-language.instructions.md`
- `specs/add-target-language.instructions.md` — the procedure for a new study language
- `specs/docker.instructions.md` — deployment contract
- `AGENTS.md` — invariants this fork must not break

**First target language: TBD (placeholder).** Several items below depend on this choice and cannot
be specified further until it is made.

## 1. Upstream-friendly fork (draft)

The fork tracks upstream closely. This is a constraint on every other item in this document.

- New behavior goes into new modules, new services and new config flags, not into edits of existing
  upstream files.
- Where an upstream file must change, the edit should be the smallest possible hook — a registration
  call, a flag check, a dispatch entry — with the real logic living in a new module.
- Every added feature is off by default behind a config flag, so a deployment with the flags unset
  behaves exactly like upstream.
- Rebase on upstream monthly. `upstream` is already configured as a remote
  (`https://github.com/ArtCC/freelingo.git`).
- Deployment-only configuration (ports, CPU image tags, bind addresses) lives in override files that
  upstream does not ship, never in edits to `docker-compose.yml`. See DEPLOY.md.
- Upstream's `AGENTS.md` invariants are treated as binding on this fork.

## 2. Tutor evals, before any tutor changes (draft)

No change to tutor prompts or correction behavior lands before there is a way to measure it.

- A corpus of 20–30 saved learner conversations, each with its mistakes annotated: what was wrong,
  where, and what the correction should be.
- Checks are code, not an LLM grading itself. Draft check list:
  - was the annotated error actually corrected
  - did the reply stay in the target language
  - is the vocabulary at the intended CEFR level, checked against a frequency list for the target
    language
- Runs are repeated, because the model is sampled and a single run is noise. Report per-check pass
  rates across repeats, not one transcript.
- Open question: where the frequency list for the chosen target language comes from, and its licence.

## 3. Multilingual TTS (draft)

Kokoro's bundled voices are English-only, which makes the bundled local TTS useless for any other
study language. Upstream's answer is to switch `TTS_PROVIDER=openai`.

- Goal: a local or self-hosted TTS path that speaks the chosen target language.
- Must slot into upstream's existing TTS provider contract
  (`specs/speech-services.instructions.md`) as an additional provider, not as a rewrite of the
  existing ones.
- Open question: which engine, and whether it runs on CPU at acceptable latency on this host — the
  deployment host has no GPU (see DEPLOY.md).

## 4. Structured corrections into the existing flashcards (draft)

- Alongside the chat reply, the tutor returns a structured list of corrections, each with: the
  learner's original text, the corrected form, and an error type.
- The structure is returned as its own field, not parsed back out of the reply prose.
- Corrections feed the SM-2 flashcard scheduler that upstream already has, so a mistake becomes
  something that comes back for review.
- Open question: the error-type taxonomy, and whether every correction should become a card or only
  those above some threshold.

## 5. Role-play scenarios from real situations (draft)

- Scenarios authored from actual situations the learner expects to be in, rather than generic
  textbook settings.
- Each scenario sets a role for the tutor, a goal for the learner, and a success condition.
- Open question: whether scenarios are user-authored content in the database or configuration
  shipped with the fork.

## 6. Family accounts (draft)

- Several learners in one household, each with their own progress, under one deployment.
- Upstream's hosted-service concerns — quotas, trials, Stripe, freemium gating — are out of scope
  for this fork and should be left switched off rather than modified.
- Open question: whether this is a new grouping above the existing user model or just several
  independent accounts with shared administration.

## Non-goals

- Running this as a hosted service for anyone outside the household.
- Changing upstream's auth, refresh-token rotation, or storage rules — `AGENTS.md` marks these as
  not to be weakened.
- Any change to the Stripe, quota, trial or freemium code beyond disabling it by configuration.

## Licence note

FreeLingo is AGPL-3.0. This deployment serves one household. The AGPL's network-use provision is the
reason the fork stays public at `sangmank/freelingo` rather than being kept private.
