# Preview PDF Fixer

Fix PDFs that look correct in Chrome, Acrobat, or Overleaf, but render broken in macOS Preview, Safari, Quick Look, or other Apple PDFKit-based viewers.

This is especially useful for figures exported from Figma and then embedded into LaTeX papers. Figma often exports drop shadows, masks, blurs, image clips, gradients, and opacity stacks as PDF transparency features. Those standalone figure PDFs can render fine. After `pdfTeX` embeds them into a compiled paper, though, Preview may show gray rectangles, missing images, rectangular masks, or weird drop shadows.

`preview-pdf-fixer` rewrites the PDF into a simpler form Apple’s renderer handles reliably.

## What Goes Wrong?

The problematic files are usually not "corrupt" in the everyday sense. They often pass `qpdf --check`, and Adobe Acrobat renders them correctly.

The fragile ingredients are PDF features such as:

- soft masks (`/SMask`)
- transparency groups (`/Group /S /Transparency`)
- blend/opacity graphics states
- SVG/Figma filters converted into image masks
- gradients, shadings, and nested Form XObjects

The failure tends to appear when a complex figure PDF is embedded inside another PDF, for example:

```tex
\includegraphics[width=\linewidth]{figures/title.pdf}
```

Chrome and Acrobat have robust renderers for this. Apple Preview/Safari can be pickier, especially with nested transparency from Figma exports inside `pdfTeX` output.

## The Fix

The safest fix is to flatten transparency.

For PDF input, the default mode uses Ghostscript `pdfwrite` to create a PDF 1.3 file. PDF 1.3 predates live transparency, so Ghostscript resolves the masks and transparency into normal page content. Text and many vector elements usually remain vector, while fragile transparent parts become renderer-friendly.

After Ghostscript rewrites the visual content, `preview-pdf-fixer` restores the original link annotations with PyPDF2. This matters for LaTeX papers: Ghostscript may otherwise drop or rewrite some clickable references, especially URL annotations on pages that get heavily rasterized.

For SVG input, the default mode rasterizes the SVG into a PDF page. This is intentionally blunt: complex SVG filters can turn back into the same soft-mask mess if converted as vector PDF. Rasterizing makes the output robust before LaTeX ever sees it.

There is also a `rewrite` mode that keeps modern PDF transparency and is smaller, but it is less bulletproof for Preview.

## Install

Clone the repo and put `bin/` on your `PATH`:

```bash
git clone https://github.com/FrankFundel/preview-pdf-fixer.git
cd preview-pdf-fixer
export PATH="$PWD/bin:$PATH"
```

Or call the script directly:

```bash
./bin/preview-pdf-fixer --help
```

### Dependencies

For PDF input:

- Ghostscript (`gs`)
- Python 3
- PyPDF2

For SVG input in default `flatten` mode:

- `rsvg-convert`
- Python 3
- Pillow

For SVG input in `rewrite` mode:

- `rsvg-convert` or Inkscape

On macOS with Homebrew:

```bash
brew install ghostscript librsvg
python3 -m pip install PyPDF2 Pillow
```

## Usage

Fix a compiled paper PDF:

```bash
preview-pdf-fixer -o paper_preview.pdf paper.pdf
```

Fix every PDF in a figure directory:

```bash
preview-pdf-fixer --out-dir fixed figures/*.pdf
```

Fix Figma SVG exports before putting them into LaTeX:

```bash
preview-pdf-fixer --out-dir fixed figures/*.svg
```

Increase SVG rasterization resolution:

```bash
preview-pdf-fixer --svg-scale 2 --out-dir fixed figures/*.svg
```

Try the smaller, less aggressive rewrite mode:

```bash
preview-pdf-fixer --mode rewrite -o paper_rewrite.pdf paper.pdf
```

Skip link restoration if you only care about visual output:

```bash
preview-pdf-fixer --no-preserve-links -o paper_visual_only.pdf paper.pdf
```

By default, an input named `paper.pdf` produces `paper_preview.pdf` beside it.

## Which File Should I Fix?

Usually, fix the compiled PDF first:

```bash
preview-pdf-fixer -o paper_preview.pdf paper.pdf
```

This is the easiest and was the most reliable path in the motivating case.

If you want the LaTeX source itself to produce a Preview-safe PDF without post-processing the whole paper, fix the figure source instead:

```bash
preview-pdf-fixer --out-dir latex/figures_safe latex/figures/title.pdf
preview-pdf-fixer --out-dir latex/figures_safe figma_exports/*.svg
```

Then include those fixed PDFs from LaTeX.

## Diagnosing the Issue

You can inspect a PDF for suspicious transparency features with:

```bash
strings paper.pdf | rg '/SMask|/Group|/Transparency|/Shading|/Pattern|/ca |/CA '
```

This is not a perfect validator, but it is a quick smell test. If the broken file contains lots of soft masks and transparency groups, flattening is likely to help.

You can also ask `qpdf` to check syntax:

```bash
qpdf --check paper.pdf
```

Passing `qpdf --check` does not mean Preview will render the file correctly. It only means the PDF is structurally valid enough for qpdf.

## Trade-Offs

- `flatten` is the safest mode for Apple Preview/Safari.
- `flatten` can make PDFs larger.
- Some transparent/vector regions may become image-like internally.
- PDF link annotations are restored by default, but unusual annotation types may not be preserved.
- `rewrite` keeps files smaller and more editable, but may leave soft masks that Preview can still mishandle.
- SVG `flatten` is rasterized by design. Use `--svg-scale 2` or higher for print-quality figure PDFs.

## License

MIT
