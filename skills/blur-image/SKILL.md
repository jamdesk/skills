---
name: blur-image
description: "Use when the user wants to blur, redact, or anonymize parts of an image — screenshots with API keys, emails, PII, customer data, or any sensitive text. Also triggers on 'hide text in screenshot', 'redact image', 'blur sensitive', 'anonymize screenshot', 'prepare screenshot for sharing', or privacy-related image editing. Use this skill even if the user just says 'blur this' with an image file."
---

# Blur Image

Detect and blur sensitive regions in screenshots using AI vision + ImageMagick.

**Announce:** "I'm using the blur-image skill to identify and blur sensitive regions."

**Use when:** Blur, redact, anonymize, hide text in screenshots, prepare images for sharing, remove PII from images.

**Skip when:** Crop, resize, format conversion, color adjustments, or non-privacy image editing.

## Critical Rules

| Always | Never |
|--------|-------|
| Check `which magick` before running; if missing, provide install instructions and stop | Run blur without user confirmation |
| Show the generated command before executing | Overwrite the original file without `--overwrite` flag |
| Use sigma >= 15 for privacy-sensitive content (low sigma is reversible) | Use sigma < 10 (can be reversed with deblurring algorithms) |
| Save output as WebP with `-quality 85` | Blur decorative/non-sensitive content without asking |
| Add 20-30px padding around detected text regions (AI coordinates have error margins) | Trust raw AI coordinates without padding |
| Read output image after blurring to verify correct regions were blurred | Skip the verification step |
| Get image pixel dimensions with `magick identify` before generating coordinates | Assume coordinates without checking actual image dimensions |

## Flags

| Flag | Behavior |
|------|----------|
| (none) | Full workflow: preflight, auto-detect, confirm, blur, verify |
| `--guided` | Skip auto-detection, ask user what to blur |
| `--overwrite` | Replace original file instead of creating `-blurred` copy |
| `--solid` | Use `-fill black -colorize 100` instead of Gaussian blur (more secure — can't be reversed) |
| `--dry-run` | Generate and show the command but don't execute it |

## Phase 1: Preflight

Check that ImageMagick is installed and the image is valid.

1. Run `which magick`. If missing, show install instructions and **stop**:
   - macOS: `brew install imagemagick`
   - Ubuntu/Debian: `sudo apt install imagemagick`
   - Other: https://imagemagick.org/script/download.php
2. Verify the image file exists and format is supported (png, jpg, jpeg, webp, tiff)
3. Get actual pixel dimensions: `magick identify -format '%wx%h' image.png`
4. Report dimensions to confirm the coordinate space — Retina/HiDPI screenshots may be 2x logical resolution

## Phase 2: Identify Regions

Two modes depending on flags and user input.

### Mode A — Auto-detect (default)

1. Read the image using multimodal vision
2. Identify regions containing sensitive text:
   - Email addresses
   - API keys, tokens, secrets
   - URLs with credentials or internal hostnames
   - Personal names, phone numbers
   - IP addresses
   - File paths with usernames
   - Passwords, database connection strings
3. For each region, determine the bounding box `(x, y, width, height)` in pixels from **top-left origin** (ImageMagick's coordinate system)
4. **Add 20-30px padding** to all sides of each detected region — AI spatial detection has 20-50px error margins
5. Present findings:
   ```
   I found N regions with sensitive text:
   1. [description] at roughly (X, Y) — WxH (with padding)
   2. [description] at roughly (X, Y) — WxH (with padding)
   ```
6. Ask user to confirm which regions to blur
7. If no sensitive content found, report that and offer `--guided` mode

### Mode B — User-guided (`--guided` or when user specifies what to blur)

1. User describes what to blur: "blur the email in the top right" or "blur everything below the header"
2. Read the image, locate the described content
3. Propose padded coordinates and ask user to confirm

## Phase 3: Generate Command

Build the ImageMagick command with all confirmed regions.

**Gaussian blur (default):**
```bash
magick input.png \
  -region WxH+X+Y -blur 0x20 \
  -region WxH+X+Y -blur 0x20 \
  -quality 85 output-blurred.webp
```

**Solid fill (`--solid` flag):**
```bash
magick input.png \
  -region WxH+X+Y -fill black -colorize 100 \
  -region WxH+X+Y -fill black -colorize 100 \
  -quality 85 output-blurred.webp
```

- Default output filename: `<original-name>-blurred.webp` alongside the original
- Present the full command and output path before running
- Ask: "Run this command? The blurred image will be saved as [path]."

## Phase 4: Execute and Verify

1. Run the command
2. Read the output image to verify blur was applied to the correct regions
3. If any text is still visible or wrong area was blurred, offer to adjust coordinates (increase padding or shift position) and re-run
4. Report success with the output file path

## Example

```
User: blur the sensitive stuff in screenshot.png

Claude: I'm using the blur-image skill to identify and blur sensitive regions.

Preflight: magick found at /opt/homebrew/bin/magick (v7.1.2)
Image: screenshot.png (1920x1080 PNG)

I found 2 regions with sensitive text:
1. Email address "user@company.com" at roughly (430, 210) — 320x65px (with padding)
2. API key "sk_live_..." at roughly (80, 560) — 390x60px (with padding)

Blur both regions? [Y/n]

User: Y

Running:
  magick screenshot.png \
    -region 320x65+430+210 -blur 0x20 \
    -region 390x60+80+560 -blur 0x20 \
    -quality 85 screenshot-blurred.webp

✓ Done. Saved to screenshot-blurred.webp
I've verified the output — both regions are blurred and surrounding content is intact.
```

## Red Flags

Stop if you catch yourself:
- Running blur without showing the command first
- Using sigma < 10 (text may be recoverable with deblurring)
- Overwriting the original without `--overwrite` flag
- Blurring the entire image instead of specific regions
- Not reading the output image to verify the blur
- Using coordinates without adding padding
- Skipping `magick identify` to confirm image dimensions
- Not handling Retina/HiDPI resolution differences
