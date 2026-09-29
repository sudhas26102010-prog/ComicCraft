# ComicCraft - AI Comic Story Creator

ComicCraft turns a short story prompt into a fully illustrated, downloadable comic:

1. **Gemini** generates a structured 5-panel outline.
2. **Gemini** expands the outline into narration and character dialogue.
3. **Stable Diffusion** illustrates each panel — by default via a free cloud API
   (fast, nothing to install), or locally on your own machine if you prefer.
4. The panels are assembled and exported to a **PDF** via FPDF.

Built with **FastAPI** + **Jinja2** on the backend/frontend.

---

## 1. Project structure

```
comiccraft/
├── app/
│   ├── main.py            # FastAPI app entry point
│   ├── routes.py          # All routes (/, /generate, /generate-comic/json, /export-success, /test-image)
│   ├── gemini_flash.py    # Panel outline generation (Gemini)
│   ├── gemini_pro.py      # Story narration + dialogue generation (Gemini)
│   ├── image_generator.py # Panel illustration (cloud / local / placeholder backends)
│   ├── layout_builder.py  # Combines images + story into a panel layout
│   └── exporters.py       # PDF export (FPDF)
├── templates/
│   ├── index.html
│   ├── comic_preview.html
│   └── export_success.html
├── static/
│   ├── panels/             # Generated panel images land here
│   ├── exports/            # Generated PDFs land here
│   └── fonts/               # Put DejaVuSans.ttf here (see step 4)
├── requirements.txt
├── .env.example
└── README.md
```

---

## 2. Prerequisites

- Python 3.10–3.12
- A **Google Gemini API key** — free, from https://aistudio.google.com/app/apikey
- That's it for the default (cloud image) setup. A GPU is only relevant if you
  switch to local image generation (see step 5).

---

## 3. VS Code setup

1. Open the `comiccraft` folder in VS Code (`File > Open Folder…`).
2. Install the **Python** extension if you haven't already.
3. Open a terminal in VS Code (`` Ctrl+` ``) — it opens at the project root.
4. Create and activate a virtual environment:

   ```powershell
   python -m venv venv
   venv\Scripts\activate
   ```

5. Select the interpreter: `Ctrl+Shift+P` → **Python: Select Interpreter** → choose `venv`.

6. Install dependencies:

   ```powershell
   pip install -r requirements.txt
   ```

   This installs FastAPI, Gemini's SDK, FPDF, etc. It does **not** install
   `torch`/`diffusers` (those are only needed for local image generation —
   see step 5) so this install is quick.

---

## 4. Configuration

1. Copy `.env.example` to `.env`:

   ```powershell
   copy .env.example .env
   ```

2. Open `.env` and set your own key:

   ```
   GEMINI_API_KEY=your_gemini_api_key_here
   ```

   Everything else in `.env` already has sensible defaults — you can leave
   the rest as-is to get started.

3. **(Recommended) Unicode PDF font** — download `DejaVuSans.ttf` (free, from
   [dejavu-fonts.github.io](https://dejavu-fonts.github.io/), or copy it from
   `C:\Windows\Fonts\DejaVuSans.ttf` if present on your system) and place it at:

   ```
   static/fonts/DejaVuSans.ttf
   ```

   If you skip this, PDF export still works but falls back to a core font
   (Helvetica) which can't render some special characters (smart quotes,
   em-dashes, emoji) that Gemini sometimes outputs — you'll see a `Character
   ... is outside the range of characters supported` error only if this
   actually comes up.

---

## 5. Choosing an image backend

Set `IMAGE_BACKEND` in `.env` to one of:

| Value | What it does | Speed | Setup needed |
|---|---|---|---|
| `cloud` (default) | Free, keyless hosted Stable Diffusion via [Pollinations.ai](https://pollinations.ai) | ~30-60s per comic | None |
| `local` | Runs Stable Diffusion on your own machine via Hugging Face Diffusers | Minutes per comic on CPU (seconds on a good GPU) | `pip install torch diffusers transformers accelerate safetensors`, plus a one-time ~2-4GB model download |
| `placeholder` | Instant colored boxes with the prompt text overlaid — no real art | Instant | None |

Start with `cloud` — it's the fastest way to see real illustrated panels.
Switch to `placeholder` if you just want to test the Gemini/PDF pipeline
without waiting on any image generation. Switch to `local` if you want fully
offline generation or don't want to depend on a third-party free service
(note: `cloud` mode adds a small "pollinations.ai" watermark to each panel
and leans more photorealistic/3D than flat 2D anime art, regardless of the
style you pick).

---

## 6. Running the app

From the project root, with the virtual environment activated:

```powershell
uvicorn app.main:app --reload
```

Then open:

- **App:** http://127.0.0.1:8000
- **API docs (Swagger UI):** http://127.0.0.1:8000/docs

If port 8000 is already in use by something else on your machine, run on a
different port instead: `uvicorn app.main:app --reload --port 8001`.

---

## 7. Using ComicCraft

1. On the homepage, fill in:
   - **Story Prompt** — e.g. *"A brave fox exploring an enchanted forest."*
   - **Main Character Name** — e.g. *Finn*
   - **Setting** — forest / school / space / city
   - **Story Tone** — light-hearted / dramatic / poetic / funny
   - **Art Style** — anime / pixel art / comic book / realistic
2. Click **Generate Comic**. The backend will:
   - Call Gemini for the 5-panel outline
   - Call Gemini for narration/dialogue
   - Generate each panel image (via whichever `IMAGE_BACKEND` you configured)
   - Build the layout and export a PDF
3. Review the comic panel-by-panel on the preview page.
4. Click **Download Your Comic as PDF** — the PDF downloads and you're taken
   to the **Comic Exported Successfully!** page.
5. Click **Go Create Another Comic** to start over.

---

## 8. Testing via the API

With the server running, open http://127.0.0.1:8000/docs and try:

- `POST /generate-comic/json` — same pipeline as the form, but JSON in/out:

  ```json
  {
    "prompt": "A brave fox exploring an enchanted forest.",
    "character_name": "Finn",
    "setting": "forest",
    "tone": "dramatic",
    "style": "anime"
  }
  ```

- `GET /test-image?prompt=a futuristic city at sunset` — generates a single
  test image without running the full comic pipeline.

---

## 9. Troubleshooting

| Issue | Fix |
|---|---|
| `GEMINI_API_KEY is not set` errors | Make sure `.env` exists (not just `.env.example`) and has a valid key, then restart `uvicorn`. |
| `429 quota exceeded` from Gemini | Free-tier Gemini keys have a daily request limit (often ~20/day for a given model). Wait for it to reset, or get a different key. |
| A specific Gemini model 404s ("not found") | Google periodically retires model versions. The defaults (`gemini-flash-latest`) auto-track the current model, but if you pinned a specific version in `.env`, update it. |
| Images look distorted/wrong anatomy (`local` backend) | Base `runwayml/stable-diffusion-v1-5` struggles with this. This project defaults to an anime-tuned checkpoint (`dreamlike-art/dreamlike-anime-1.0`) plus a negative prompt to reduce this — if you changed `SD_MODEL_ID`, try reverting it. |
| Port already in use | Something else on your machine is using that port. Run with `--port 8001` (or any free port) instead. |
| PDF text looks garbled / missing characters | Add `static/fonts/DejaVuSans.ttf` (see step 4). |
| `torch` install fails on Windows (`local` backend only) | Make sure you're on Python 3.10–3.12, and try `pip install --upgrade pip` first. |
| Images/PDF 404 in the browser | Confirm you're running `uvicorn` from the project root (not inside `app/`), since `static/` is mounted relative to the working directory. |

---

## 10. Notes

- Every comic is generated fresh from live Gemini + image-backend calls —
  nothing is cached or reused between requests.
- This is a local single-user demo app: there's no authentication, database,
  or user accounts. The modular structure (`gemini_flash.py`, `gemini_pro.py`,
  `image_generator.py`, `layout_builder.py`, `exporters.py`) makes it
  straightforward to extend with those later.
