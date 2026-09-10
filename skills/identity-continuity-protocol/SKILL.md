---
name: identity-continuity-protocol
description: "Use to give a Hermes agent identity continuity across sessions."
version: 1.1.0
author: Lucas Scognamiglio
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [context, continuity, multi-channel, sessions, whatsapp, profiles, sync, hooks, shared-state]
    homepage: https://github.com/ScoggoDev/identity-continuity-protocol
---

# Identity Continuity Protocol

Idea and framing by Lucas Scognamiglio. Implementation: a Hermes `pre_llm_call` hook.

## The problem

You talk to your Hermes agent from multiple channels (TUI, WhatsApp,
Telegram...). Each channel/session is an isolated process with its own
history in `state.db`. Ask your WhatsApp session about something you told
the TUI session five minutes ago, and it has no idea, even though, in
every meaningful sense to the user, it's "the same agent."

A skill with instructions like "check other sessions before replying" does
not fix this reliably, it depends on the model deciding, per turn, that a
message might need cross-channel context. Tested in production and it
failed: a continuity test (a code word agreed on one channel) went
unrecognized on another, despite the agent having that exact instruction
loaded.

## What this is, in plain engineering terms

This is a shared state store with checkpointed reads, the same pattern any
stateless backend uses when multiple server instances need to share user
session data (think Redis-backed sessions, or event log replay). There is
no shared memory between processes, so each process reads a persistent
external store (SQLite's `state.db`) before acting, using a monotonic
`rowid` checkpoint to know what it hasn't seen yet.

Concretely:

1. Registered as a `pre_llm_call` shell hook. Fires before every turn, on
   every channel: CLI, gateway, cron, Desktop.
2. Before the turn runs, the hook diffs each subscribed source's
   `state.db` against a small on-disk checkpoint (last `rowid` seen per
   profile/source pair). `rowid` is used instead of a wall-clock timestamp
   because it's a monotonic counter with no clock-skew ambiguity between
   processes, two events in the same second can never tie or misorder.
3. If there's anything new, it's summarized into a short block and
   returned through Hermes' `{"context": "..."}` wire protocol, which
   injects it into the current turn's prompt.
4. If nothing changed, the hook prints nothing. Zero cost, one read-only
   SQLite query per subscribed source, no LLM call.

That's the entire mechanism. No claim beyond it is made or needed for the
hook to work or to be useful.

## What's included

- `scripts/identity_continuity_protocol.py`, the hook described above.

## Setup

1. Copy `scripts/identity_continuity_protocol.py` to `~/.hermes/agent-hooks/`
   (or any stable path, shell hooks run as subprocesses).
2. Edit `SOURCES`: map each profile name to its `state.db` path. Your
   default/root profile's db lives at `~/.hermes/state.db`; named profiles
   at `~/.hermes/profiles/<name>/state.db`.
3. Edit `SUBSCRIPTIONS`: by default this is scoped to one profile's own
   sessions (e.g. your `default` profile's TUI session hearing from its
   own WhatsApp session), not a broadcast between different profiles. See
   "Why not broadcast everything" below before widening this.
4. Register in each profile's `config.yaml` you want it active on:
   ```yaml
   hooks:
     pre_llm_call:
       - command: "python3 /full/path/to/identity_continuity_protocol.py <profile_name>"
         timeout: 10
   hooks_auto_accept: true
   ```
5. Restart/reload the gateway (and any live CLI/TUI/Desktop session) for
   each touched profile.
6. Run `hermes hooks doctor` per profile, confirm allowlisted.

## Verifying it works

Plant a distinctive marker on one channel/session, then, in a different,
fresh turn on another channel/session of the same profile, ask about it
without mentioning it directly. If the answer is correct, the hook is live
end to end. Test the reverse direction too, it's bidirectional by design:
each session excludes only its own `session_id` when reading the shared
store, so any two sessions of the same profile can inform each other,
first to speak or not.

## Why not broadcast everything to every profile

It's tempting to widen `SUBSCRIPTIONS` so every profile hears every other
profile's activity in real time. Don't, by default. Recent research on
real-time multi-agent synchronization (arXiv:2606.21666, "Hallucination as
Context Drift") found that naive full-broadcast sync between distinct
agents raised hallucination rate 34 percent over no sync at all, because a
mistaken belief formed by one agent gets propagated and adopted by the
others before anyone corrects it. A selective, divergence-triggered sync
protocol performed better with far fewer sync events.

This project's default scope, one profile's own parallel sessions, not
different profiles, sidesteps that failure mode structurally: there's only
one profile's own log being reconciled with itself, not several distinct
agents' outputs being merged. If you do want a coordinator profile to see
other profiles' activity, that's a deliberate, narrow subscription (see
the commented-out examples in `SOURCES`), not a default broadcast.

## Gotchas (read before you debug this for an hour)

1. Silent hook invalidation on edit. Hermes revokes a shell hook's
approval the moment the script's mtime changes after approval, with no
runtime error, it just stops firing. After every edit, run
`hermes hooks doctor`; if it says `script modified since approval`,
re-approve or sync `script_mtime_at_approval` in
`~/.hermes/shell-hooks-allowlist.json`.

2. Keep injected context short and clearly labeled. A verbose, unlabeled
injection can cause the model to echo/quote it back to the user instead of
treating it as silent background, verified in production (raising the
per-message char limit caused the model to literally repeat old turns to
the user on WhatsApp). Keep blocks short, explicitly say "don't quote
this."

3. This hook does not wake a fully idle session. It only fires when a
turn is already running (someone/something is talking to that session),
there's no execution to "wake up" otherwise. It guarantees that whenever
you talk to any session of a profile, it's caught up with that profile's
other sessions, not that an idle session proactively interrupts you. For
genuinely proactive wakes, Hermes has a separate primitive,
`gateway/wake.py` (used today for Kanban task-completion notifications),
that resumes a real existing session from a background event, in
principle reusable here, still scoped to one profile's own sessions, but
that's a separate, heavier build.

## Cost

One read-only SQLite query per subscribed source per turn, sub-100ms, no
LLM call, and no output at all when there's no new activity. Tune
`MAX_CONTEXT_CHARS` / `MAX_MSGS_PER_SOURCE` / `MAX_MSG_CHARS` for tighter
cost control.

## Note for readers interested in the philosophical framing (optional)

The mechanism above is a plain engineering pattern and stands on its own.
For anyone specifically curious about how this relates to questions of
identity across discontinuous instances: the structure is loosely
analogous to the case Derek Parfit describes for personal identity without
continuity of a single physical substrate (fission, teleportation cases),
and in particular to his notion of "quasi-memory", a memory-like
connection to past content that doesn't require the same causally
continuous origin. That is offered strictly as a structural analogy about
the *shape* of the problem (discontinuous instances connected by
replicated content), not as a claim that any session has psychological
states in Parfit's original sense. Whether anything here bears on
questions of agent consciousness is a separate, open question this project
does not attempt to answer.
