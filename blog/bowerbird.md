---
layout: default
title: "bowerbird: Turn a Slide Deck into a Narrated Video, in Your Own Voice"
date: 2026-10-29
description: "A small internal tool for endless pitching: write the script, set the length, and get a narrated deck video in your own cloned voice — so you can fail fast, fail often, and ship the next version the same day."
tags: [tools, video, voice-cloning, local-first, founders]
categories: [product, tools]
---

# bowerbird: Turn a Slide Deck into a Narrated Video, in Your Own Voice

*A small tool I built for myself. Sharing it in case it helps you too.*

---

## Who It's For

People who pitch constantly and need each pitch to go out as a video:

- **Founders** pitching an idea, a product, or a new angle — over and over, to different audiences
- **Engineers and PMs** pitching a design, a proposal, or a demo to people who weren't in the room
- **Anyone** testing which version of a story actually lands

The way to find the pitch that works is to fail fast and fail often: make one, send it, see what lands, change it, send the next one. That only works if each version is cheap to make.

If your video is really *a deck plus a clear explanation*, this is for you. If you need B-roll, cuts to camera, or anything cinematic, it isn't.

---

## The Problem

The deck is usually done. Turning it into a video is where the time goes — and when every version costs that much, you make fewer versions.

- **Recording takes many takes.** Getting every slide right in one pass rarely happens. A stumble on slide 9 means starting over, and the retake of slide 3 is often worse than the first.
- **Fixing it means editing.** Cutting fillers, trimming silences, lining audio up with slide changes. That's a separate skill and a separate app.
- **Length is a guess.** You aim for three and a half minutes and find out it's six only after recording.
- **Small changes are expensive.** One updated number on one slide means re-recording, or re-editing, the whole thing.

A few-minute pitch video ends up costing an evening. Without a faster way, iterating was slow and fewer pitches went out — which means fewer chances to fail, learn, and fix the story.

---

## What bowerbird Does

[bowerbird](https://github.com/akshayeKS/bowerbird) takes your slide deck and a written script and produces a narrated video in your own cloned voice, entirely on your machine.

```
deck.pdf / .pptx / .html ──┐
                           ├── bowerbird init ──> slides/*.png + script.json
 target length (3:30) ─────┘
                                      │
                         ✎ you write the narration
                                      │
 samples/*.wav ── bowerbird voice ──> cloned profile
                                      │
                                      ▼
                            bowerbird build ──> deck.mp4
```

```bash
bowerbird init deck.pdf --target 3:30   # slides to images, script.json scaffold
bowerbird draft                         # optional: prompt to draft narration with any LLM
bowerbird voice                         # clone your voice from a few short samples
bowerbird check                         # tools, voice, script, and length vs target
bowerbird build                         # -> deck.mp4
```

You record your voice once — a few 20–30 second samples — and after that every video is just text.

---

## How It Addresses Each Pain

**No takes.** The narration lives in one file, `script.json`, one entry per slide. You fix a sentence by editing text, not by performing it again.

**No editing.** Pacing is a number per slide (`gap_after`, in seconds). Silence trimming, loudness, and syncing audio to slides are handled for you, and slide timings are checked against the final audio so nothing drifts.

**Length is known up front.** You give a target length, and bowerbird gives you a character budget. `check` tells you where you'll land before anything is generated:

```
  ok    length               3:28.40 vs target 3:30 (-2s) @ 17.4 chars/s
```

After each build it re-measures your actual speaking rate, so estimates get more accurate over time.

**Small changes stay small.** Audio is generated and cached per slide. Change one slide, regenerate one slide:

```bash
bowerbird build --slides 7,12    # redo just these
bowerbird build --video-only     # adjust pacing, re-render in about a minute
```

**More pitches, faster.** A new version of the pitch is a script edit and a rebuild, not a recording session. That's what makes failing fast actually affordable: try a new opening, a different order, a shorter cut — and get it out the same day.

**It doesn't oversell.** If a slide is marked Planned, Roadmap, or Beta, `check` warns when the narration never says so. It's easy to present roadmap as shipped when you're writing to a time limit.

---

## Privacy

It's your voice, so it stays with you.

- Voice cloning runs locally via [Voicebox](https://voicebox.sh). Nothing is uploaded.
- The only network call is the optional `draft --llm`, which you have to ask for explicitly.
- No Python dependencies — standard library only. Open source, MIT.

On a work laptop, check with your IT or security team before installing Voicebox. And if the deck is confidential, so is the script.

---

## Try It

```bash
git clone https://github.com/akshayeKS/bowerbird
cd bowerbird && pip install -e .

bowerbird init examples/demo-deck/deck.html --target 3:30
bowerbird check   # tells you exactly what's missing
```

Works with HTML, PDF, and `.pptx`/`.odp` decks (Keynote: export to PDF first). Needs Python 3.11+ and ffmpeg.

If you try it and something breaks, or it's missing what you need, DM me on [X (@akshayeKS)](https://x.com/akshayeKS).

---

*Your deck, your voice.*

---

[← Back to Blog](/blog) ・ [Home](/) ・ [Contact](/contact)
