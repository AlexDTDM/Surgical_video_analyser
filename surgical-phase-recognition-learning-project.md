# Surgical Phase Recognition — Learning Project

A first, deliberately simple project: take a frame from a cholecystectomy video and predict **what is happening** — the surgical phase, and which instruments are in view — then turn that prediction into a plain-language sentence. The goal is to learn the full machine-learning pipeline end to end on real, clinically meaningful data, with a surgeon available to sanity-check the output.

---

## 1. Purpose

This is a **learning project first, product second**. The aim is to go from raw surgical video to a human-readable "what's happening" statement using the simplest approach that genuinely works, and to understand every step along the way: data, a pretrained model, a trained head, evaluation, and a grounded output.

It is intentionally the first slice of a much larger pipeline (event recognition → structured timeline → operative report), so nothing learned here is wasted if the project is ever scaled up — but none of that complexity is required now.

## 2. What you'll build

A small system that, given a single video frame:

1. predicts the **surgical phase** (1 of 7),
2. predicts which **instruments** are present (multi-label, 7 tools),
3. composes a **template sentence** describing the frame, e.g.
   *"The gallbladder is being dissected using a hook and a grasper."*

No video/temporal modeling, no text-generation model, no report structure. Frame in → labels out → templated sentence.

## 3. Scope

**In scope**

- Frame-level phase classification (single label).
- Frame-level tool presence detection (multi-label).
- A deterministic template that maps predictions to a sentence.
- Evaluation against ground-truth labels, plus an informal surgeon check.

**Deliberately deferred (not now)**

- Temporal modeling — frames are treated independently; no smoothing, LSTM, or TCN yet.
- Action triplets ⟨instrument, verb, target⟩.
- Operative-report structure or narrative generation.
- LLM-based phrasing (the template is enough to start).
- Robotic video, kinematics, multi-procedure generalization.

Each deferred item is a natural *next* lesson, listed in §10.

## 4. Dataset

**Cholec80** — 80 laparoscopic cholecystectomy videos, fully labeled. It is the standard entry point for surgical video AI, which means abundant baselines and documentation to learn from.

- **Phases (7, single-label per frame):** preparation, Calot's triangle dissection, clipping and cutting, gallbladder dissection, gallbladder packaging, cleaning and coagulation, gallbladder retraction.
- **Tools (7, multi-label per frame):** grasper, bipolar, hook, scissors, clipper, irrigator, specimen bag.
- **Sampling:** videos are 25 fps; subsample to ~1 fps to keep the dataset manageable and because adjacent frames are near-duplicates.
- **Split:** the canonical split is 40 videos for training and 40 for testing. Hold out a few training videos (e.g. 8) as a validation set for tuning.
- **Access:** via the CAMMA request form at https://camma.unistra.fr/datasets/.
- **License note:** research / non-commercial use — fine for a learning project; just don't ship anything commercial off it without checking terms.

## 5. Approach

Keep the model as small and standard as possible.

- **Backbone:** a pretrained image model (e.g. a torchvision ResNet) used as a **frozen** feature extractor. No need to train it from scratch or even fine-tune it at first.
- **Phase head:** a small classifier (one or two linear layers) on top of the frozen features, trained with cross-entropy over the 7 phases.
- **Tool head:** a second small head trained with binary cross-entropy over the 7 tools (multi-label, so each tool is an independent yes/no).
- **Sentence:** a lookup table from phase → verb phrase and tool → noun, assembled into one sentence. Fully grounded in the prediction, so there is nothing to hallucinate.

Why this is the right first design: the backbone already knows generic visual features, so the only thing you train is a lightweight head — fast, cheap, and easy to reason about. It runs on modest hardware and even on CPU for small experiments.

## 6. Build steps

Each step is a milestone with a clear "done when," and each teaches one thing.

### Step 1 — Familiarize (no code)
- **Do:** watch one video alongside its phase labels; have the surgeon narrate a real case so the 7 phases become concrete rather than abstract strings.
- **Learn:** what the phases actually look like, where they're ambiguous, and what "tool present" really means on screen.
- **Done when:** you can look at a random frame and guess the phase yourself.

### Step 2 — Zero-training baseline
- **Do:** run a generic pretrained image model (or an off-the-shelf vision-language model) on a handful of frames and observe how poorly it captures surgical specifics.
- **Learn:** *why* surgical-specific training is necessary — you feel the gap instead of taking it on faith.
- **Done when:** you have concrete examples of generic models getting it wrong.

### Step 3 — Train the phase head
- **Do:** build the data loader (1 fps frames + phase labels), freeze the backbone, train the phase classifier head, and evaluate on held-out videos.
- **Learn:** the core training loop — batching, loss, optimization, overfitting, train/val/test discipline.
- **Done when:** test accuracy is clearly above chance and you can plot predicted-vs-true phase across a full held-out video and watch the timelines line up.

### Step 4 — Add tools, then sentences
- **Do:** add the multi-label tool head; combine phase + detected tools into the template sentence; show a sample of sentences to the surgeon.
- **Learn:** multi-label vs single-label outputs, and how to turn predictions into grounded language.
- **Done when:** you can feed in a new frame and get a believable sentence, and the surgeon agrees it's roughly right.

## 7. Evaluation

- **Phase head:** overall accuracy, per-phase accuracy, and a confusion matrix (which phases get mixed up — usually adjacent ones).
- **Tool head:** per-tool precision/recall or average precision (tools are imbalanced — some appear rarely).
- **Qualitative:** the predicted-vs-true **phase timeline** for a held-out video is the most intuitive check; jittery predictions are the visible motivation for temporal smoothing later.
- **Human check:** the surgeon reviews a sample of generated sentences for plausibility. This is informal — a reality check, not a formal study.

## 8. Surgeon in the loop

The one surgeon is most valuable in two narrow moments, not as a constant reviewer:

- **At the start (Step 1):** narrating a case to ground your understanding of the phases and tools.
- **At the end (Step 4):** spot-checking generated sentences against what actually happened.

Keep their time cheap and high-leverage; the rest of the work is self-directed.

## 9. Suggested repo structure

```
surgical-phase-recognition/
├── data/                # Cholec80 (gitignored — licensed data)
├── notebooks/
│   └── 01_explore.ipynb # watch frames + labels (Step 1)
├── src/
│   ├── dataset.py       # frame sampling + label loading
│   ├── model.py         # frozen backbone + phase head + tool head
│   ├── train.py         # training loop
│   ├── evaluate.py      # metrics + timeline/confusion plots
│   └── sentence.py      # phase/tool → template sentence
├── README.md
└── requirements.txt
```

## 10. Stretch goals (the next lessons)

In rough order of difficulty, each a deliberate follow-up:

1. **Temporal smoothing** — post-process the per-frame predictions (e.g. majority vote over a window) and watch the timeline clean up.
2. **A sequence model** — feed frame features into an LSTM or temporal convolution to model order directly.
3. **Fine-tune the backbone** — unfreeze and train end-to-end; compare against the frozen baseline.
4. **Action triplets** — move from "what phase" to ⟨instrument, verb, target⟩ using CholecT50.
5. **LLM phrasing** — replace the template with an LLM that turns the structured prediction into more natural prose.

## 11. Practical notes

- **Environment:** Python, PyTorch, torchvision; pandas/matplotlib for evaluation plots.
- **Compute:** a single GPU is comfortable; small experiments run on CPU. Because the backbone is frozen, you can precompute and cache its features once and train the heads in seconds — a big time-saver while learning.
- **Common gotchas:** frame extraction at the right rate; aligning phase/tool annotation files to sampled frames; class imbalance (some phases and tools are rare); and never letting test videos leak into training.
- **Reminder:** keep the dataset out of version control — it's licensed.

---

*This project is the first slice of a larger vision (event recognition → structured timeline → operative report). Built simply and end to end, it teaches the whole pipeline in miniature while leaving every harder piece for a later, deliberate step.*
