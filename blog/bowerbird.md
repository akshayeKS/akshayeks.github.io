---
layout: default
title: "bowerbird: Narrated Deck Videos Without the Editing Loop"
date: 2026-10-29
description: "I kept going back and forth trying to make every recording crisp. bowerbird turns a slide deck into a narrated video in my own voice — time-boxed, re-recordable per slide, and fully local. Good enough until I'm a full-time video editor."
tags: [tools, video, voice-cloning, local-first, founders]
categories: [product, tools]
---

# bowerbird: Narrated Deck Videos Without the Editing Loop

*Time-box it. Record it. Re-record it. Ship it.*

---

## The To-and-Fro

I make a lot of short deck videos — product walkthroughs, investor updates, demos for people who can't make the call. Three to five minutes, a handful of slides, me talking over them.

And every single one turned into the same loop:

1. Record the whole thing.
2. Stumble on slide 6. Say "um" four times on slide 9.
3. Re-record. Now slide 3 sounds worse than the first take.
4. Open an editor. Cut, nudge, trim silences, line audio up with slide changes.
5. Realize it's 6 minutes, not 3:30. Go back to step 1.

The goal was always *crisp*. What I got was an evening gone, for a video people watch once at 1.5x.

I'm not a video editor. I'd love to get good at it someday. But until I'm doing that full time, I needed a setup that does two things: **keeps me inside a time limit** and **lets me redo the one bad part without touching the rest.** With as little gear and as few apps as possible.

---

## So I Built bowerbird

[bowerbird](https://github.com/akshayeKS/bowerbird) turns a slide deck into a narrated video, in your own cloned voice, entirely on your machine.

The bowerbird builds an elaborate display to present with, and mimics the sounds it hears. That's the whole tool: your deck, your voice.

```
deck.pdf / .pptx / .html ──┐
                           ├── bowerbird init ──> slides/*.png + script.json
 target length (3:30) ─────┘
                                      │
                         ✎ you write the narration     ← the part that matters
                                      │
 samples/*.wav ── bowerbird voice ──> cloned profile
                                      │
                                      ▼
                            bowerbird build ──> deck.mp4
```

```bash
bowerbird init deck.pdf --target 3:30   # rasterize slides, scaffold script.json
bowerbird draft                         # prompt to draft narration with any LLM
bowerbird voice                         # clone your voice from a few samples
bowerbird check                         # tools, voice, script, and length vs target
bowerbird build                         # -> deck.mp4
```

The minimalist setup: a laptop, 3–6 voice samples of 20–30 seconds each recorded in a quiet room, ffmpeg, and a local [Voicebox](https://voicebox.sh) install. No mic rig, no timeline editor, no retakes of the whole thing.

---

## Why This Instead of Just Recording

### The script is the product

When I record live, the words, the delivery, and the timing all have to be right at the same moment. That's why it takes ten takes.

bowerbird separates them. Everything is plumbing around one editable file — `script.json`, one entry per slide. Fix a sentence by editing text, not by re-performing it. Every "um" I used to cut out by hand simply never exists. (In fact, the drafting rules say never write "um" — TTS pronounces it as a word, and it sounds worse than no filler at all.)

### A time limit you find out about *before* rendering

This was the biggest one for me. My first draft written to a 3:30 target came in at **6:09 — 44% over** — and I couldn't tell by reading it.

Now `init` gives you a character budget for your target length, and `check` tells you where you'll land before a single clip is generated:

```
  ok    length               3:28.40 vs target 3:30 (-2s) @ 17.4 chars/s
```

After each build, bowerbird re-measures your actual speaking rate and saves it, so the next estimate is sharper. The time limit stops being a hope and becomes a number you write to.

### Re-record one slide, not the whole video

Clips are generated and cached per slide. One line sounds off? Redo just that one:

```bash
bowerbird build --slides 7,12    # re-roll those two, leave the rest alone
bowerbird build --video-only     # tweak pacing, re-render in ~1 min, no TTS
```

This is the exact thing the to-and-fro was missing. Re-recording used to cost me the whole video. Now it costs one slide.

### Pacing is declared, not edited

Each slide carries a `gap_after` in seconds — about 0.3 mid-thought, 0.5–0.65 at real transitions. That's the whole "edit."

An earlier version tried to detect slide boundaries from the audio with silence detection. It was fragile, needed tuning, and once starved a four-word sentence down to 0.35 seconds of screen time. A number in a file is better.

---

## The Bits I Learned the Hard Way

A few rules are baked into the drafting prompt, each from getting it wrong:

- **Don't read the slide.** The viewer can read. Say what the slide doesn't.
- **Write one continuous argument.** Each slide picks up where the last left off — *"And here's the scale…"*, *"But…"* — otherwise you get N announcements instead of a story.
- **Spell numbers out.** "eight hundred sixty", not "860".
- **Say when something isn't shipped yet.** If a slide says Planned, Roadmap, or Beta, `check` warns when the narration never hedges. Writing to a time budget makes it very easy to quietly present roadmap as shipped. I'd rather sound a bit stilted than oversell.

And a couple of unglamorous ffmpeg details that make the output actually usable: loudness is normalized once across the whole video (per-clip normalization flattens delivery), only head and tail silence is trimmed (the pauses between sentences are breathing, not defects), and audio is pinned to 48 kHz so Slack, Drive, and Keynote don't choke on it.

---

## On Privacy

It's your voice. That's biometric data, and I didn't want it on anyone's server.

- Voice cloning runs against Voicebox on `localhost`. Nothing is uploaded.
- The only network call bowerbird makes is the optional `draft --llm`, which you have to ask for by name. Without it, `draft` just writes a prompt you can paste wherever you like.
- No Python dependencies — standard library only.
- Open source, MIT — [read every line](https://github.com/akshayeKS/bowerbird).

If you're on a work laptop, check with your IT/security team before installing Voicebox. And remember that if your deck is confidential, your narration script is too.

---

## Try It

```bash
git clone https://github.com/akshayeKS/bowerbird
cd bowerbird && pip install -e .

bowerbird init examples/demo-deck/deck.html --target 3:30
bowerbird check   # tells you exactly what's missing
```

Works with HTML, PDF, and `.pptx`/`.odp` decks (Keynote: export to PDF first). You need Python 3.11+ and ffmpeg.

---

## Until I'm a Full-Time Video Editor

bowerbird isn't trying to replace real video editing. It won't do B-roll, cuts to camera, or anything cinematic. It's for the 80% of videos that are really just *a deck and a clear explanation* — where the time I'd spend getting it crisp is better spent on the thing the deck is about.

Set the time. Write the script. Record. Re-record the one slide that's off. Done.

If you make deck videos and know the to-and-fro, give it a spin and tell me what breaks. DM me on [X (@akshayeKS)](https://x.com/akshayeKS).

---

*Your deck, your voice. Less editing, more shipping.*

---

[← Back to Blog](/blog) ・ [Home](/) ・ [Contact](/contact)
