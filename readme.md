# smart-pdf-md

Windows batch script to **mass-convert PDF (.pdf) to Markdown (.md)** with smart routing:

* **Textual PDFs →** fast text extraction via **PyMuPDF** (fitz)
* **Scanned/non-textual PDFs →** robust conversion via **Marker** (`marker-pdf` / `marker_single`), with automatic **page slicing** to avoid OOM/timeouts

---

## Why “smart”?

`smart-pdf-md.bat` inspects each PDF. If enough pages contain real text, it uses PyMuPDF (much faster). Otherwise it falls back to Marker’s high‑quality PDF→Markdown path. Very large PDFs are processed in **slices** (configurable, default 40 pages) to increase reliability.

---

## Features

* **Batch/recursive conversion** of one file or an entire folder tree
* **Auto‑install** Python dependencies on first run (`pymupdf`, `marker-pdf`)
* **Heuristic routing** between fast text extraction and Marker OCR/layout path
* **Slice processing** for large docs with progressive backoff on errors
* **Inline logging** with consistent tags (`[scan]`, `[file]`, `[path]`, `[OK]`, `[WARN]`, `[ERROR]`…)
* **Zero setup** beyond having Python + pip on PATH (Windows)
* **UTF‑8 console** (uses `chcp 65001`)

> **Note:** Image extraction in the Marker path is **disabled by default**; output focuses on Markdown text.

---

## Requirements

* **Windows** (runs in `cmd.exe` via `.bat`)
* **Python** with **pip** available on PATH

  * The script checks/prints Python version/bitness and pip version
* Internet access on first run (to install `pymupdf` and `marker-pdf` if missing)
* Optional: **CUDA‑capable GPU** (the script exports `TORCH_DEVICE=cuda` for Marker). If you don’t have a compatible GPU/driver, switch to CPU (see **Configuration**).

---

## Quick Start

1. Place `smart-pdf-md.bat` anywhere on your system (or clone/download the repo).
2. Open **Command Prompt** in the folder containing your PDFs, or pass a path.
3. Run:

```bat
smart-pdf-md.bat [INPUT] [SLICE]
```

**Arguments**

* `INPUT`
  Path to a **PDF file** or a **folder**. If omitted, defaults to the **current directory**.
* `SLICE`
  Max pages per Marker slice. Default **40**. The script halves this (down to a min of 5) on failures and retries.

**Examples**

```bat
:: Convert all PDFs recursively under the current folder (default slice=40)
smart-pdf-md.bat

:: Convert one folder, larger slices (50 pages)
smart-pdf-md.bat "D:\Docs\Handbooks" 50

:: Convert a single file
smart-pdf-md.bat "C:\Reports\2024\survey.pdf"
```

**Output location**

* For each `input.pdf`, the tool writes `input.md` **next to** the PDF (same folder).

---

## What you’ll see (logs)

Representative log tags:

```
[boot] smart_pdf_md.bat starting...
[cfg ] Input            : "C:\Docs"
[cfg ] Slice pages      : 40
[cfg ] Output format    : markdown
[cfg ] Image extraction : DISABLED
[cfg ] DPI (low/high)   : 96 / 120
[lint] Python OK       : 3.11 / 64-bit
[lint] pip OK          : pip 24.x
[deps] PyMuPDF present.
[deps] marker-pdf present.
[env ] TORCH_DEVICE=cuda
[env ] OCR_ENGINE=surya
[io  ] Writing driver: "%TEMP%\smart_pdf_md_driver.py"
[scan ] folder: C:\Docs  files=37
[file ] (1/37) C:\Docs\foo.pdf
[path ] TEXTUAL -> fast PyMuPDF
[TEXT ] C:\Docs\foo.pdf -> C:\Docs\foo.md  (0.42s)
[file ] (2/37) C:\Docs\scanned.pdf
[path ] NON-TEXTUAL -> marker_single
[MRK_S] total_pages=240 slice=40 dpi=96/120
[RUN  ] marker_single ...
[OK   ] pages 0-39 in 35.22s
...
[done] smart_pdf_md.bat finished.
```

**Exit codes** (driver)

* `0` success
* `1` input path not found
* `2` slice processing failed even at minimum slice size
* `3` Marker single‑pass failed when PDF could not be opened by PyMuPDF
* `9` unhandled error

---

## Configuration

The batch file sets a few knobs you may want to change.

### Marker/torch environment

```
TORCH_DEVICE=cuda          # use 'cpu' if you don’t have a supported GPU
OCR_ENGINE=surya           # default OCR used by Marker
PYTORCH_CUDA_ALLOC_CONF    # tuned allocator settings for CUDA
```

> Edit these near the top of `smart-pdf-md.bat`. Setting `TORCH_DEVICE=cpu` improves compatibility at the cost of speed.

### DPI used by Marker

The generated Python driver defines:

```
LOWRES = 96
HIGHRES = 120
```

These influence Marker’s internal rendering during conversion. Increase for higher fidelity (slower) or decrease for speed. Edit the constants inside the **driver generation** section of the `.bat` if needed.

### Slice size

Pass `SLICE` on the command line (default 40). On failures, the driver halves the slice (`40 → 20 → 10 → 5`) and retries. Minimum slice is 5.

### Image extraction

The Marker path is configured for **Markdown text only** by default. If you want embedded images, call `marker_single` manually with the appropriate flags, or adapt the command in the batch file (search for the lines that build the `marker_single` command in the generated driver).

---

## How it works (under the hood)

1. **Toolchain check**: verifies `python`/`pip` and prints versions.
2. **Deps**: ensures `pymupdf` and `marker-pdf` are installed (installs if missing).
3. **Env**: exports Marker‑related env vars (device, OCR engine, CUDA allocator).
4. **Driver emit**: writes a temporary **Python driver** to `%TEMP%` and **pre‑compiles** it (`py_compile`).
5. **Routing** per file:

   * **Textual detection**: opens with PyMuPDF and counts pages with ≥100 non‑whitespace chars; if ≥20% of pages qualify → **fast text extraction** path writes a single `.md` by concatenating page text with blank lines.
   * Otherwise → **Marker path**:

     * If the PDF can’t be opened by PyMuPDF, run a **single‑pass** `marker_single`.
     * If it opens, process in **slices** of `SLICE` pages using `marker_single` per slice, shrinking the slice on errors.
6. **Output**: `.md` written next to the PDF.

---

## Best practices & tips

* **No GPU?** Set `TORCH_DEVICE=cpu` in the `.bat` to avoid CUDA initialization errors.
* **Huge PDFs**: increase `SLICE` only if you have ample RAM/VRAM; otherwise keep or lower it.
* **Speed vs quality**: PyMuPDF is far faster but only extracts text; layout and tables are preserved better by Marker.
* **Stuck conversions**: lower `SLICE`, lower `DPI`, or switch to CPU if CUDA is unstable.

---

## Troubleshooting

* **“Python not found on PATH”**: Install Python from python.org or Microsoft Store and select *Add python.exe to PATH*.
* **pip install failures**: Ensure internet access and run the `.bat` from an elevated prompt if your environment requires it.
* **CUDA errors**: Set `TORCH_DEVICE=cpu` in the `.bat`, or ensure a compatible NVIDIA driver/CUDA runtime is present.
* **Garbled output (encoding)**: The script sets `chcp 65001` for UTF‑8; ensure your console font supports the glyphs.
* **Empty Markdown for scanned PDFs**: That indicates the fast path was chosen but the doc was actually scanned. Raise the heuristic (see below) or force Marker by temporarily disabling the fast path in the driver.

### Adjusting the “textual” heuristic (advanced)

Inside the generated driver, the function `is_textual(pdf, min_chars_per_page=100, min_ratio=0.2)` controls routing. You can raise `min_chars_per_page` or `min_ratio` to send more borderline documents through Marker.

---

## Contributing

PRs for:

* Optional image extraction toggle and output directory control
* Configurable heuristics via CLI flags
* Robust CPU/GPU auto‑detection for Marker
* Unit tests and CI

---

## License

Check out `license.md`.

---

## Appendix: Command reference (summary)

```
Usage: smart-pdf-md.bat [INPUT] [SLICE]

INPUT  : PDF file or directory (recursive). Default = current directory.
SLICE  : Max pages per slice for Marker. Default = 40. Min = 5 (auto‑backoff).

Output : Writes <filename>.md next to each PDF.
Return : 0=OK, 1=not found, 2=slice failed, 3=marker single‑pass failed, 9=unhandled.
```
