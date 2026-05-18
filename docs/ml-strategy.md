# Keepy — ML Strategy

## MVP Scope: Dominoes Only

The MVP focuses exclusively on Dominoes. Phase Ten card recognition will be added in a later version once the core scoring pipeline is proven and stable.

This decision keeps the ML problem well-defined for the first release:
- One game type, one visual domain
- One model to train, evaluate, and improve
- Faster iteration on the full app before expanding

---

## Base Model: Roboflow Universe — Dominoes Dataset

**Dataset source:** [https://universe.roboflow.com/bruce-rund-a5msv/dominoes-gjo45](https://universe.roboflow.com/bruce-rund-a5msv/dominoes-gjo45)

This dataset provides labeled images of domino tiles with class labels representing pip counts. The model detects individual tile values (number of dots), which maps directly to the scoring logic needed for Keepy — no additional parsing or transformation required.

The base model was trained by the dataset author and is available for download in YOLOv8 PyTorch format (`.pt`).

---

## Model Architecture: YOLOv8

Keepy uses **YOLOv8** (You Only Look Once, version 8) via the [Ultralytics](https://docs.ultralytics.com) library for object detection.

YOLOv8 was chosen because:
- It is fast enough for real-time inference on CPU (no GPU required for MVP)
- The Ultralytics Python API is clean and well-documented
- It supports fine-tuning from an existing checkpoint, which is exactly the workflow used here
- The Roboflow dataset exports directly in YOLOv8-compatible format

The nano variant (`yolov8n`) is used as the starting point — smallest model size, fastest inference, sufficient accuracy for well-lit game photos.

---

## Hosting Strategy: Self-Hosted

The model runs **locally within the FastAPI backend**. There is no external inference API call at runtime.

**Why self-hosted:**
- No per-inference charges — costs are flat and predictable
- No dependency on a third-party API that could change pricing, go down, or impose rate limits
- Full control over the model version and update schedule
- During development, everything runs on a local machine at zero cost

**How it works:**
- The trained `.pt` weights file is stored in the repository (or a designated model directory)
- The FastAPI ML service loads the model into memory once at server startup
- Inference runs in-process — the mobile app sends an image, the model processes it, the result is returned

---

## Training Plan

### Phase 1 — Use the base model as-is
Download the pre-trained weights from the Roboflow dataset. Run inference on real domino photos taken during actual gameplay to evaluate real-world accuracy.

### Phase 2 — Fine-tune on real game photos
Collect photos taken with the app during testing — different lighting conditions, tile styles, table surfaces, and angles. Label them using Roboflow or LabelImg and run additional training epochs on top of the base weights.

This is called **transfer learning** — the model already knows how to detect domino pips, fine-tuning just makes it better at the specific conditions your users will encounter.

### Phase 3 — Evaluate and iterate
Track the model's confidence scores in production. If a high percentage of snapshots are returning low-confidence flags (below 0.80), that signals the model needs more training data for those conditions.

---

## Scoring Logic

The model returns a list of detections. Each detection has:
- A **class label** — the pip value detected (e.g. `"3"`, `"5"`)
- A **confidence score** — how certain the model is (0 to 1)

The scoring function:
1. Filters out detections below a minimum confidence threshold (e.g. 0.50) to ignore noise
2. Sums all detected class values to produce the round score
3. Computes an overall confidence as the average (or minimum) confidence across all detections
4. If overall confidence is below **0.80**, returns a `low_confidence: true` flag alongside the score

The player sees the calculated score and can override it if it is wrong.

---

## Future: Phase Ten

When Phase Ten support is added, the same pipeline applies:
- Find or create a labeled dataset of Phase Ten cards
- Fine-tune a YOLOv8 model on that dataset
- Add a separate scoring function that applies Phase Ten point values (cards 1–9 = 5 pts, cards 10–12 = 10 pts, Skip = 15 pts, Wild = 25 pts)
- The ML service selects which model to run based on the room's game type

---

## File Structure

```
keepy-backend/
├── ml/
│   ├── models/
│   │   └── dominoes_best.pt     # trained YOLOv8 weights
│   ├── inference.py             # model loading + inference logic
│   └── scoring.py               # game-specific scoring functions
```

---

## Dependencies

| Package       | Purpose                                      |
|---------------|----------------------------------------------|
| ultralytics   | YOLOv8 model loading and inference           |
| Pillow        | Image preprocessing before inference        |
| numpy         | Array operations on detection results       |
