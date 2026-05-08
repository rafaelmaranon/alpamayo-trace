# alpamayo-trace

Live demo: [rafaelmaranon.github.io/alpamayo-trace](https://rafaelmaranon.github.io/alpamayo-trace/)

**VLA ≠ VLM.** A driving model and a vision model, watching the same San Francisco intersection.

A self-contained, single-page viewer that runs **NVIDIA Alpamayo R1** (a vision-language-action model) alongside **Qwen2.5-VL-72B** (a general-purpose vision-language model) on the same 44-second SF dashcam clip at 5 Hz, and surfaces what each model produces per frame, in parallel.

![alpamayo-trace viewer screenshot](screenshot.png)

Watch a 2-minute walkthrough on YouTube: [youtu.be/oJ1EG7fOTvE](https://youtu.be/oJ1EG7fOTvE)

## Try it yourself

Local: clone the repo and open `index.html` in Chrome or Firefox. Safari users: run `python3 -m http.server` in the repo directory and visit `localhost:8000`.

The viewer's trace list opens focused on frame 212 (t = 42.2 s), where:

- **Qwen2.5-VL (VLM)** says: *"...lane markings clear, no visible construction barriers."*
- **Alpamayo R1 (VLA)** says: *"Nudge to the right to pass the construction cone in the lane."*

The encroachment is real — it's a **restaurant parklet** (wooden outdoor seating jutting into the parking lane), not a cone. Alpamayo's action is right; its label is approximate. Qwen's denial is the larger error: an object is clearly there, and Qwen volunteered the negative without being asked.

That's the demo. Click `i` for the full caveat. Click `show list ▼` to scrub through all 220 paired reasonings + narrations.

## What the demo shows

Two distinct failure modes surfaced by running both streams in parallel.

**VLA label drift across training-prior analogs.** In a 2.6-second window from 40.2 s to 42.8 s — as a restaurant parklet enters the field of view — Alpamayo flags a lane-encroaching obstacle in six frames (202, 203, 204, 212, 213, 215). It labels the same obstacle differently each time: barrier → barricade → sign → cone → cone → line. The action is consistent (nudge right or left). The label class drifts across training-prior analogs.

The model learned the function (lane encroachment → nudge maneuver) but generalizes the label from the closest training-prior analog, and that analog can change frame-to-frame. Action right. Labels approximate and unstable.

For control, this might be fine — the right nudge happens regardless of what the model calls the obstacle. For evaluation, logging, edge-case mining, or regulator audits, it matters: a VLA that produces correct actions with confidently inconsistent labels is a different kind of partial reliability than one that fails outright.

**VLM fluent denial.** In the same window, Qwen2.5-VL explicitly asserts *"no construction barriers are visible"* or doesn't mention obstruction at all — even though something is clearly encroaching into the lane. The prompt didn't ask about construction. Qwen volunteered the negative.

Generative VLMs trained on balanced scene descriptions optimize for fluent prose. *"No construction barriers"* sounds more authoritative than the honest *"there might be small lane-side objects I can't classify."* Fluent denial fills the description; calibrated uncertainty would have cost linguistic flow.

**Both failures are interesting; they're different kinds.** A VLA can be correct in the way that matters most (action) while wrong in the way that matters for downstream use (label) — and unstably so. A VLM can be fluent and wrong simultaneously. Neither is captured by single-output evaluation. Running both streams in parallel lets you see both at once.

## Honest caveats

- **Open-loop, single clip.** This evaluates the V+L layer of a VLA on a found YouTube clip. It's not a closed-loop driving demo. The full A in VLA needs a real loop with proprioception and a controller.
- **Ego pose is recovered from optical flow** with a constant-velocity prior — wobbly and unit-arbitrary. No GPS, no IMU. Toggle the trajectory display on (off by default) to see what Alpamayo's planner produces given imperfect proprioception. Same model on a clip with real ego pose produces smooth trajectories — the wobble is the input's fault, not the model's.
- **Public Alpamayo R1 release is SFT-only.** The Alpamayo paper describes a fuller multi-stage training pipeline (SFT + RL for reasoning quality + RL for reasoning-action consistency). The publicly released 10B weights only include the SFT stage. Results here reflect the public model.
- **Six obstacle-flag frames in a 2.6-second window is a single-clip case study,** not a benchmark. The pattern is suggestive of two real failure modes (VLA training-prior label drift, VLM fluent denial), but it's not a "VLAs always do this" claim. Test it yourself; the data is here.

## How it was built

For each frame at 5 Hz over 44 seconds:

1. **Alpamayo R1 (NVIDIA, 10B-parameter VLA)** — runs on a RunPod H100, takes 4 cameras × 4 timesteps plus 16 ego-pose history points, outputs (a) a one-clause action-justification ("chain-of-causation") and (b) a 64-waypoint trajectory.
2. **Qwen2.5-VL-72B (Alibaba)** — runs on Nebius API, takes the front-camera frame and a "describe what a driver sees" prompt, outputs a ~50-word scene narration.

220 frame pairs, all inlined into this self-contained HTML.

## Repo structure

```
alpamayo-trace/
├── index.html       # Self-contained viewer with all 220 paired reasonings inlined
├── clip.mp4         # The source SF dashcam clip (44 sec, 1920×1080)
├── screenshot.png   # README header image
├── README.md        # This file
└── LICENSE          # Apache 2.0 (code only; clip credit below)
```

No backend. No build step. No dependencies.

## Background reading

Five papers shaping AV reasoning right now:

| Paper | Date | What |
|-------|------|------|
| [Wayve LINGO-2](https://wayve.ai/thinking/lingo-2-driving-with-language/) | Apr 2024 | First closed-loop VLA on public roads. Production-grade, closed source. |
| [Waymo EMMA](https://arxiv.org/abs/2410.23262) | Oct 2024 | Built on Gemini, treats every driving input/output as language tokens. |
| [Wayve GAIA-2](https://arxiv.org/abs/2503.20523) | Mar 2025 | Controllable generative world model. Generates safety-critical scenarios. |
| [NVIDIA Alpamayo R1](https://arxiv.org/abs/2511.00088) | Oct 2025 | First VLA trained on causally-linked reasoning ("Chain of Causation"). |

Three camps emerging: Wayve ships in production but locks the box. Waymo publishes ideas but keeps the weights. NVIDIA opens the whole stack and bets on the ecosystem. Each a different theory of how AV gets to scale.

## Source clip

[https://www.youtube.com/watch?v=GZH5F_dZ0Q4](https://www.youtube.com/watch?v=GZH5F_dZ0Q4) — used here under fair use for non-commercial research and education. All rights belong to the original uploader.

## License

The viewer code (HTML + CSS + JS) is released under **Apache License 2.0** — see [`LICENSE`](LICENSE).

The model outputs (Alpamayo R1 chain-of-causation traces and Qwen2.5-VL narrations) reflect the behavior of the upstream models on this specific clip. Refer to the upstream model licenses for any reuse:

- [nvidia/Alpamayo-R1-10B](https://huggingface.co/nvidia/Alpamayo-R1-10B) — non-commercial license, commercial licensing on request
- [Qwen/Qwen2.5-VL-72B-Instruct](https://huggingface.co/Qwen/Qwen2.5-VL-72B-Instruct) — Qwen License

The video clip is not redistributed under this repo's license — it is included only for demonstration and remains subject to its original YouTube terms.

Not affiliated with NVIDIA, Alibaba/Qwen, Wayve, or Waymo.

## Citation

If you reference this demo in a write-up, a link suffices. There's no paper to cite.

```
alpamayo-trace: a single-page viewer surfacing the dual-stream gap between
NVIDIA Alpamayo R1 (VLA) and Qwen2.5-VL (VLM) on a 44-second SF clip at 5 Hz.
https://github.com/rafaelmaranon/alpamayo-trace
```
