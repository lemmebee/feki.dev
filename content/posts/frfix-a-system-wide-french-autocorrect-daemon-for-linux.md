+++
date = '2026-05-19T21:00:00+02:00'
draft = false
title = 'frfix: A System-Wide French Autocorrect Daemon for Linux'
+++
**tl;dr:** Web browsers, chat apps, and IDEs all ship their own spellcheckers, except your terminal, your password-protected note app, and half the niche tools you actually use every day. `frfix` is a small Python daemon that reads keystrokes via `evdev`, runs them through `hunspell` and `grammalecte`, and silently rewrites French mistakes anywhere on Linux/X11. No extension, no clipboard trick, no input method engine. Source: [github.com/lemmebee/frfix](https://github.com/lemmebee/frfix).

---
## Table of contents
- [The Problem: Spellcheck Stops at the App Boundary](#the-problem-spellcheck-stops-at-the-app-boundary)
- [The Idea: Spellcheck That Acts Instead of Underlining](#the-idea-spellcheck-that-acts-instead-of-underlining)
- [What It Corrects](#what-it-corrects)
- [How It Works](#how-it-works)
  - [1. Reading Keystrokes with `evdev`](#1-reading-keystrokes-with-evdev)
  - [2. Translating Keycodes via X11/xkb](#2-translating-keycodes-via-x11xkb)
  - [3. Building Words and Sentences](#3-building-words-and-sentences)
  - [4. Correcting with hunspell + grammalecte](#4-correcting-with-hunspell--grammalecte)
  - [5. Injecting the Fix with `xdotool`](#5-injecting-the-fix-with-xdotool)
- [Why X11 Only (and Why Not Wayland)](#why-x11-only-and-why-not-wayland)
- [Security: Typing Secrets Must Stay Secret](#security-typing-secrets-must-stay-secret)
- [Install and Run](#install-and-run)
- [Configuration](#configuration)
- [Limitations and Honest Trade-offs](#limitations-and-honest-trade-offs)
- [What's Next](#whats-next)

## The Problem: Spellcheck Stops at the App Boundary

Writing French on Linux is a paper-cut experience. Firefox autocorrects, GNOME Text Editor underlines, Slack catches half of it. Then you switch to a terminal, a Tauri app, an Electron tool that disabled spellcheck, an SSH session, a custom React form with `spellcheck="false"`, or a GTK widget nobody bothered to wire up, and you're back to typing `francais` and `cest` and pretending nobody noticed.

Every app reinvents spellcheck. Most do it badly. Many skip it entirely. The result: you carry the same mental tax of "remember the accents" into every single text field on the system, and your muscle memory still loses.

I wanted spellcheck to be a property of the **keyboard**, not the **app**.

## The Idea: Spellcheck That Acts Instead of Underlining

`frfix` is a background daemon. It does three things in a loop:

1. Watches keystrokes system-wide via `/dev/input`.
2. Maintains a small word/sentence buffer.
3. When a word ends (space, punctuation, Enter), it checks the word/sentence against hunspell and grammalecte. If broken, it rewrites it in place: backspace × N, then type the fix.

There is no UI to interact with. There is no app integration. It does not know which window is focused beyond an exclusion check. It just sits there, watches you type, and fixes mistakes the instant the word ends, in the terminal, in `vim`, in an Electron app that disabled spellcheck, in `xterm`, in everything.

Think of it as a French-only, system-wide spellchecker that *acts* instead of underlining.

## What It Corrects

The four categories that cover ~95% of everyday French typos:

- **Missing accents**: `francais` → `français`, `realite` → `réalité`, `ecran` → `écran`, `etre` → `être`.
- **Elisions**: `cest` → `c'est`, `jai` → `j'ai`, `lhomme` → `l'homme`, `cetait` → `c'était`, `quil` → `qu'il`. The single most annoying class of typo when switching from English layouts where the apostrophe is awkward.
- **AZERTY/QWERTY swaps**: `vrqi` → `vrai`, `fqire` → `faire`, `voulqis` → `voulais`. The classic "I forgot which layout was active" failure.
- **Grammar at sentence boundaries**: `tu peut` → `tu peux`, `je veut` → `je veux`, simple agreement nits. Handled by grammalecte once the sentence terminates.

There's also an **undo**: `Ctrl+Z` immediately after a correction reverts it. A small ring buffer tracks the last few corrections so you can back out if frfix gets it wrong.

The system is **conservative by design**: if a word is already a valid French word, nothing happens. False positives are the enemy. A spellchecker that rewrites things you typed on purpose is worse than no spellchecker at all.

## How It Works

```
                ┌──────────┐    keysym +    ┌──────────┐
   /dev/input ─▶│  evdev   │───keycode ────▶│ KeyTrans │
                │ capture  │  (read-only)   │  X11/xkb │
                └──────────┘                └────┬─────┘
                                                 │ char
                                                 ▼
                              ┌───────────────────────────────┐
                              │ TextBuffer (word + sentence)  │
                              └────────────┬──────────────────┘
                                           │ word / sentence
                                           ▼
                              ┌───────────────────────────────┐
                              │ FrenchCorrector               │
                              │  • hunspell (spelling)        │
                              │  • grammalecte (grammar)      │
                              │  • user dictionary            │
                              └────────────┬──────────────────┘
                                           │ replacement
                                           ▼
                              ┌───────────────────────────────┐
                              │ Injector (xdotool)            │
                              │  backspace × N + type fix     │
                              └───────────────────────────────┘
```

### 1. Reading Keystrokes with `evdev`

Linux exposes input devices at `/dev/input/event*`. With membership in the `input` group, a process can open those files read-only and receive every key event the kernel sees, independent of X server, compositor, or focused application. `python-evdev` wraps this in an async iterator.

Critically, this is **passive capture**. There is no keyboard grab. Every other application still receives every key normally. frfix is a silent observer, not a man-in-the-middle.

### 2. Translating Keycodes via X11/xkb

`evdev` gives you raw Linux keycodes (`KEY_A`, `KEY_SEMICOLON`, …). Those are useless on their own: the same physical key produces `q` on AZERTY and `a` on QWERTY. To know what character was actually typed, frfix queries the **active xkb layout** via X11 and resolves keycode → keysym → character through the layout table.

This is what makes it correctly handle a user who switches between QWERTY and AZERTY mid-session: the same `KEY_Q` produces `q` or `a` depending on which layout xkb says is active right now.

By default, frfix only runs when the current layout is `fr`. On startup it switches to `fr` and remembers your previous layout; on exit it restores it. (You can override this with `--no-layout-check` or `--force-layout`.)

### 3. Building Words and Sentences

A `TextBuffer` keeps two pieces of state:

- The **current word**, growing one character at a time.
- The **current sentence**, accumulating across word boundaries.

Word boundaries (space, tab, punctuation, Enter) trigger spelling checks. Sentence boundaries (`.`, `!`, `?`) trigger grammar checks. Backspace shortens the current word. The buffer is reset on focus changes and on injected corrections (so frfix doesn't recursively try to "fix" its own fix).

### 4. Correcting with hunspell + grammalecte

Two layers of checking, in this order:

- **`hunspell` + `hunspell-fr-comprehensive`** for word-level checks. If the word is valid French, do nothing. If not, generate candidates with hunspell's suggestion engine, plus targeted rules for the common cases (missing accent on a known stem, missing apostrophe after `c`/`j`/`l`/`qu`/`d`/`n`/`s`/`t`, AZERTY/QWERTY substitution).
- **`grammalecte`** (optional, via GObject Introspection) for sentence-level grammar. Catches the verb-agreement nits hunspell can't see because each word in isolation is valid.

A user dictionary at `~/.config/frfix/dictionary.txt` is consulted first. Anything in there is treated as a valid French word and never corrected. Drop names, brand names, and jargon there.

### 5. Injecting the Fix with `xdotool`

When a correction fires, frfix synthesizes input events via `xdotool`:

1. `xdotool key BackSpace` × `len(original)` to delete what the user typed.
2. `xdotool type --` *replacement* to type the fix.

`xdotool` is X11-only. It speaks XTEST to the X server. The injection lands in whichever window currently has keyboard focus, exactly as if the user had typed it. The application receiving the keys cannot tell the difference between a human and `xdotool`, which is both the strength and the soft spot of the design.

## Why X11 Only (and Why Not Wayland)

Two layers of the design require capabilities Wayland deliberately removes:

- **Global keystroke capture.** Under Wayland, only the focused application receives key events. There is no equivalent to `/dev/input` reads at the protocol level (you can still do it at the device level, but layout translation breaks because there's no equivalent of `xkb` query). This is a security feature, not a bug.
- **Global keystroke injection.** Wayland has no XTEST equivalent. `xdotool` doesn't work. Tools like `ydotool` work via `/dev/uinput` but are blocked by most compositors from injecting into other applications' focused surfaces.

So `frfix` runs on **X11 only**. If you're on a GNOME-on-Wayland desktop, switch your session to Xorg at the login screen (the gear icon on the login prompt).

This isn't a temporary state. The cleanest path to Wayland support is a portal. An XDG desktop portal for "system-wide input correction" doesn't exist yet, and probably shouldn't, for the same reasons Wayland doesn't expose keylogging primitives in the first place. For now: X11.

## Security: Typing Secrets Must Stay Secret

A daemon that reads every keystroke you type is, by construction, a keylogger with extra steps. Three rules govern the design:

1. **Nothing is ever persisted.** Keystrokes, words, and sentences live in memory in a small ring, and only the last few corrections are retained (for `Ctrl+Z` undo). Nothing is written to disk. Nothing is sent over the network. There is no telemetry, no analytics, no "anonymous usage statistics."
2. **Default exclusions.** The default config excludes password managers (`keepassxc`, `1password`, `bitwarden`) by app class, and any window whose title matches `password`, `mot de passe`, or `sudo`. When the focused window matches any exclusion, frfix doesn't even buffer the keystrokes; they pass through untouched.
3. **User-extensible exclusions.** If you type secrets into an unusual app, add it to `exclusions.apps` or `exclusions.window_titles` in the config. The exclusion check happens *before* anything is buffered or compared against the dictionary.

This trust model is the cost of the design. If you can't accept a daemon with read access to `/dev/input`, don't run it. That's the same trust model as every screen-recording tool, every macro recorder, and every accessibility utility on Linux, but it deserves saying out loud.

## Install and Run

```bash
git clone https://github.com/lemmebee/frfix.git
cd frfix
bash setup.sh
```

`setup.sh` installs system packages (`hunspell`, `gobject-introspection`, python venv tooling, `xdotool`), adds your user to the `input` group, creates a virtual environment, and installs a systemd user unit. If you were just added to `input`, log out and back in before running.

Then:

```bash
# foreground, interactive
.venv/bin/frfix

# verbose: see keystrokes, candidate words, correction decisions
.venv/bin/frfix --debug

# as a systemd user service, auto-start on login
systemctl --user enable --now frfix
journalctl --user -u frfix -f
```

The `--debug` flag is the single most useful thing when something isn't working. It prints every keystroke received, every word boundary detected, and every correction decision. If words show up but corrections never fire, hunspell isn't installed or the layout check is rejecting your current layout. If no keystrokes show up at all, you're not in the `input` group (or haven't logged out and back in since being added).

## Configuration

Config lives at `~/.config/frfix/frfix.toml`, auto-created on first run:

```toml
[general]
enabled = true

[corrections]
spelling = true     # word-level: accents, typos, elisions
grammar  = true     # sentence-level grammalecte rules

[overlay]
enabled       = true
duration_ms   = 1500
bg_color      = "#1a1a2e"
text_color    = "#e0e0e0"
highlight_color = "#4ecca3"

[exclusions]
apps          = ["keepassxc", "1password", "bitwarden"]
window_titles = ["password", "mot de passe", "sudo"]
```

A small GTK overlay (toast) flashes the original → corrected pair so you actually *learn* what you mistyped, instead of frfix silently smoothing over the same typo for the 500th time. Disable it by setting `[overlay].enabled = false`.

The user dictionary (`~/.config/frfix/dictionary.txt`) is one word per line. Anything in there is treated as a valid French word. Use it for names, brand names, jargon, internal company terminology.

## Limitations and Honest Trade-offs

- **X11 only.** Already covered. Wayland is structurally not supported.
- **French only.** The whole correction layer is built around hunspell-fr and grammalecte. Multi-language support would mean detecting the language of each word, which is a hard problem and explicitly out of scope.
- **Inject-via-xdotool is timing-sensitive.** Some apps (mostly Electron and certain terminal multiplexers) occasionally swallow or reorder injected events. When it goes wrong, you get a half-corrected word; `Ctrl+Z` undoes it.
- **No semantic understanding.** Grammalecte's rules are good but not perfect. Sentences with rare constructions can trigger false positives. Add corrections to your user dictionary; if a particular grammar rule misfires repeatedly, disable `[corrections].grammar`.
- **Alpha quality.** The corrector is conservative on purpose. There are still classes of typo it doesn't catch yet, especially complex elisions (`s'il` vs `si il`) and compound words.

The goal isn't to be perfect. The goal is to fix the 95% of trivial mistakes that *every* French typist makes *every* day, so the cognitive budget goes to the actual content.

## What's Next

The pieces I'm most interested in extending:

- **Better candidate scoring.** Currently the first plausible hunspell suggestion wins. A frequency-weighted scorer over a French corpus would do better.
- **Per-app behavior.** Disable grammar in terminals (where you're typing commands, not prose), enable strict mode in editors.
- **Statistics.** Local-only. What classes of mistake does the user make most? Strictly opt-in, never leaves the machine.

If you write French daily on Linux and the per-app spellcheck inconsistency annoys you the way it annoys me, give it a try. PRs welcome.

Source: [github.com/lemmebee/frfix](https://github.com/lemmebee/frfix). MIT.
