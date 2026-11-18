# Local oMLX council

This branch adds a local, browser-based implementation of the council flow to the
original `model-council` reference project:

```text
one brief
   │
   ├── parallel independent answers from three local seats
   │
   ├── human checkpoint
   │      └── optional blind peer review of shuffled answers
   │
   └── active chairman synthesis, with or without peer reviews
```

The stages are intentionally separate. The first stage creates independent evidence;
the human decides whether extra review is worth the latency and compute; the final
chairman actively selects, weights, and rejects evidence instead of averaging it into a
vague compromise.

## Analysis and adaptation

Much credit goes to [kfc283-sudo/model-council](https://github.com/kfc283-sudo/model-council)
and its MIT-licensed implementation. Its central design is the right part to preserve:

1. Give every seat the same brief at the same time.
2. Keep the first answers independent and show disagreement instead of hiding it.
3. If the question deserves more scrutiny, shuffle and relabel the answers before
   asking reviewers to score strengths, weaknesses, and rankings.
4. Let a chairman make an explicit selection and explain the agreement and disagreement.
5. Keep a human in the loop at the point where additional review becomes a cost/benefit
   decision.

This branch changes the transport and interaction model, not that reasoning pattern:

- **Transport:** the browser talks to the small local server in
  `council/omlx_server.py`, which calls oMLX's OpenAI-compatible `/v1` API.
- **Seat composition:** the model catalog is read from `/v1/models`, so the UI can use
  whatever MLX-format chat/VLM models are installed rather than a hard-coded provider
  registry. Qwen, LLaMA, Mistral, Gemma, DeepSeek, MiniMax, GLM, and other installed
  families can be selected by their oMLX model IDs.
- **Reasoning display:** explicit `reasoning_content`/`reasoning` fields and inline
  `<think>...</think>` output are normalized into a collapsible thinking panel. The
  visible answer stays separate from the reasoning details.
- **Vision:** an optional image is sent as an OpenAI-compatible `image_url` data part.
  The selected oMLX model must be a VLM; a text-only model will return its own upstream
  capability error.
- **Privacy:** prompts, answers, images, and reviews are kept in memory for the current
  run. The browser stores only the draft, model choices, settings, and the last eight
  question briefs in localStorage. The server does not write a council transcript.

The implementation deliberately leaves the reviewer-to-answer mapping out of the
review response. Reviewers receive labels such as Answer A/B/C, and the chairman sees
the same anonymous labels. Reviewer provider names are not shown in the UI or forwarded
to the chairman.

## Installation manual

### 1. Requirements

- Apple Silicon Mac running macOS 15 or newer for oMLX.
- Python 3.10 or newer for this repository's local web server.
- oMLX installed and a model directory containing at least three usable chat models;
  include a VLM if image prompts are needed.
- A modern browser.

oMLX itself is documented at [omlx.ai](https://omlx.ai/) and in the
[official oMLX repository](https://github.com/jundot/omlx). Its current quickstart
uses a local OpenAI-compatible server at `http://localhost:8000/v1`:

```bash
# Managed server
omlx start

# Or a foreground server with an explicit model directory
omlx serve --model-dir ~/models
```

The oMLX server must be running before the model council can discover models.

### 2. Start this project

From the repository root:

```bash
python -m council.omlx_server
```

Open [http://127.0.0.1:8765](http://127.0.0.1:8765) in the browser. The local council
server proxies requests to `http://127.0.0.1:8000` by default.

For a different oMLX address or local UI port:

```bash
OMLX_BASE_URL=http://127.0.0.1:8000 COUNCIL_PORT=8765 python -m council.omlx_server
```

PowerShell equivalent:

```powershell
$env:OMLX_BASE_URL = "http://127.0.0.1:8000"
$env:COUNCIL_PORT = "8765"
python -m council.omlx_server
```

If the oMLX server requires authentication, set `OMLX_API_KEY`. In local proxy mode
the key is used only by the Python proxy. In static/direct mode the API key can instead
be entered in the browser's **oMLX** settings panel; it is stored in that browser's
localStorage and sent as a Bearer token to the configured oMLX server.

No additional Python package is required by the local council server; it uses the
standard library. The original Bedrock command path still requires its documented
`boto3` setup.

## GitHub Pages and Windows access

The `web/` folder is also a dependency-free static site. The branch includes a GitHub
Actions workflow at `.github/workflows/pages.yml` that publishes that folder to GitHub
Pages. Once Pages is enabled with **GitHub Actions** as the publishing source, the fork's
project site is normally available at:

```text
https://dtywong.github.io/model-council-omlx-local/
```

A Windows user does not need Python or a local checkout to use that site. Open the site,
click **oMLX** in the top bar, and enter:

1. **Server address** — the reachable oMLX server root, for example
   `https://mac.example.com:8000`. The UI adds `/v1` for the OpenAI-compatible API.
2. **API key** — optional; sent as `Authorization: Bearer …` when supplied.

For this browser-to-server mode, configure oMLX and the network so that:

- oMLX listens on a reachable interface, not only `127.0.0.1` on the Mac hosting it;
- the Windows machine can reach the address through the same LAN, a VPN, or a secure
  tunnel;
- the HTTPS GitHub Pages site calls an HTTPS oMLX endpoint (a plain HTTP endpoint will
  normally be blocked as mixed content); and
- oMLX CORS permits the exact Pages origin, ideally `https://dtywong.github.io`, rather
  than exposing an unauthenticated wildcard endpoint.

Do not forward an unauthenticated oMLX port directly to the public Internet. Prefer a
VPN, an authenticated HTTPS reverse proxy, or a private tunnel. If you run the Python
server locally instead, leave the settings address empty and open
`http://127.0.0.1:8765`; the UI then uses the local proxy and keeps the API key on the
machine running that proxy.

The public source and GitHub Pages artifact intentionally do not contain a server
address or API key from a private environment file. The **oMLX** settings remain visible
and editable so each user can enter their own connection. Never solve convenience by
committing a real API key: a browser-delivered key is inspectable even when the input is
masked.

## User manual

### Stage 1 — independent seats

1. Start oMLX and refresh the model catalog if the page was already open.
2. Choose three distinct models. You can select discovered models or paste a custom
   oMLX model ID into a seat.
3. Enter one question or decision brief. The draft is saved locally in the browser.
4. Enable or disable **Deep thinking mode** and set the thinking budget and answer limit.
5. Optionally attach one image for a VLM seat.
6. Select **Ask the council**. All three seats receive the same brief in parallel.
7. Watch the processing banner: it shows that the council is actively processing and,
   in direct browser mode, how many seats have returned. Local proxy mode shows an
   animated waiting state because the proxy returns the combined council response only
   after all seats finish.
8. Compare the three columns. Expand **Deep thinking returned** and **Run details** when
   the model/server provides those fields.

### Human checkpoint

Read the independent answers before invoking the next stage. This is the deliberate
human-in-the-loop decision point:

- Stop after Stage 1 when the answers are sufficiently clear.
- Run Stage 2 when the question is consequential, answers disagree, or you want an
  explicit adversarial check.
- Run Stage 3 directly when you want a chairman to select among the independent answers
  without peer review.

### Stage 2 — optional anonymized peer review

1. Choose two distinct reviewer models.
2. Select **Run anonymous review**.
3. The server strips provider names from the answer packet, securely shuffles the answer
   order, and labels the packets A/B/C.
4. Two reviewer seats independently assess every visible packet for strengths,
   weaknesses, errors, a 1–10 score, and a ranking.
5. Inspect the anonymous packet and both reviews before continuing.

The review stage is not a vote. It is evidence for the chairman and a way to surface
omissions or incorrect confidence.

### Stage 3 — active chairman synthesis

1. Choose the chairman model.
2. Select **Ask the chairman** when you are ready.
3. The chairman receives the anonymized answer packet and, if Stage 2 ran, the review
   text. It must provide a clear recommendation plus a council readout describing
   agreement, disagreement, and how it resolved the disagreement.
4. Treat the synthesis as decision support. For high-impact decisions, verify claims
   independently and keep the final human decision with the user.

### Local storage and privacy

The browser's localStorage contains the current draft, selected model IDs, thinking
settings, up to eight recent question briefs, and—only if configured in the static
settings panel—the oMLX server address and API key. It intentionally does not store
model answers, reasoning, review packets, or images. Use **Clear local history** to
remove the saved question history on this browser/device; remove the oMLX settings
manually if you no longer want them retained. The local server also does not persist
the run.

## Troubleshooting

| Symptom | Check |
|---|---|
| `oMLX unavailable` | Confirm oMLX is running and that `OMLX_BASE_URL` points to its server root, not a path that already ends in `/v1`. |
| No models discovered | Confirm models are installed in oMLX's configured model directory, then press the refresh button. |
| A seat fails | Check the model ID, available unified memory, and the oMLX error shown in that column. |
| Image request fails | Use a VLM model for that seat and keep the image at or below the local 8 MB limit. |
| Reasoning is empty | The selected model may not return reasoning, or deep thinking may be disabled. Visible answer text is still usable. |
| Browser cannot connect | Open the exact local council URL printed by `python -m council.omlx_server`; port 8765 is the default. |
| GitHub Pages cannot connect | Check the oMLX address, HTTPS/mixed-content rules, CORS, firewall/VPN reachability, and the API key. |
| Processing appears stuck | Model requests time out after 180 seconds (model discovery after 15 seconds); check oMLX logs, model loading, the thinking budget, selected chat/VLM models, and network reachability. |

## Files added by this branch

- `council/omlx_server.py` — local static server, oMLX proxy, stage orchestration, and
  reasoning/image normalization.
- `web/index.html`, `web/styles.css`, `web/app.js` — the three-column council UI and
  localStorage-backed notebook.
- `docs/omlx-local-council.md` — this analysis, installation manual, and user manual.
