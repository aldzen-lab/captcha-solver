# aliyun — Aliyun Captcha 2.0 slide-puzzle solver

Harvest-only solver for Aliyun Captcha 2.0 (the slide-puzzle used by Qoder et al).
No third party (no CapMonster). Renders the widget on a minimal self-hosted page,
detects the gap with OpenCV, drags the slider along the empirically-fitted **quadratic**
handle→piece curve, and harvests the SDK-built token from `captchaVerifyCallback`.

## Request

```json
POST /solve
{
  "type": "aliyun",
  "scene_id": "1r7eif79x",   // required — target site's captcha SceneId
  "prefix": "13lbkb5",       // required — captcha-open endpoint prefix
  "region": "sgp",           // optional — sgp (default) | cn | intl
  "proxy": "http://user:pass@host:port",  // optional
  "timeout_s": 90            // optional
}
```

No `sitekey` and no `url` — Aliyun's challenge identity is `scene_id` + `prefix`, and
the solver hosts its own minimal page (CapMonster-style; never visits the target site).

## Response

```json
{
  "type": "aliyun",
  "solved": true,
  "token": {
    "sceneId": "1r7eif79x",
    "certifyId": "nVk57gcoC0",
    "deviceToken": "SG_WEB#...",
    "data": "JRMnbQ9RGiMx..."
  },
  "verify_code": "T001",
  "method": "quadratic-slide",
  "attempts": 6,
  "elapsed": 8.4
}
```

The caller replays `token` **immediately** into `VerifyCaptchaV3` (server returns T001).
The token is **session-bound + one-time-use**; `deviceToken` is time-bound. If a proxy
was used to solve, run the verify from the same IP.

## How it works (RE notes)

1. **Popup render** — the widget only pops from a clean page; the target's heavy SPA
   suppresses it. We mount the SDK on our own minimal page with the site's sceneId.
2. **Gap detection** — `gap_cv.py`: cv2 Sobel-x gradient template match (piece
   silhouette edges vs back-image edges). Pure CPU, ~5-20ms, no LLM, no server load.
3. **Quadratic drag (the anti-bot trick)** — handle→piece is NOT linear:
   `piece_rel = 0.00355·hx² + 0.0769·hx` (residual ≈0). Naive linear solvers always
   miss (F015). We invert the quadratic to get the exact drag distance.
4. **Server verdicts** — F002 = no slide, F015 = wrong position, F001 = close, T001 =
   pass. Used as ground truth; the solver retries (challenge refresh is free) until T001.

## Status / known limitation

Fundamentally works — reaches **T001** (server-verified). The limiter is gap-detection
reliability: ~50% single-shot on high-contrast backgrounds, lower on busy ones, so the
solver leans on internal retry. On a bad streak it can exhaust `max_attempts` without a
T001 (returns `solved:false, last_verify_code`). To make it production-tight, improve
`gap_cv.detect_gap_x` (the drag math is already correct):

- multi-scale / notch-shape template match, or a tiny trained gap detector
- the error is content-dependent (not a fixed bias), so a constant offset won't fix it

Vision-LLM localization was tested and **does not work** for this (0/6) — LLMs are good
at discrete tile classification, not pixel-precise gap localization. Keep it CV-based.
