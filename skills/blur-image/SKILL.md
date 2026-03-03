---
name: blur-image
description: "Use when the user wants to blur, redact, or anonymize parts of an image — screenshots with API keys, emails, PII, customer data, or any sensitive text. Also triggers on 'hide text in screenshot', 'redact image', 'blur sensitive', 'anonymize screenshot', 'prepare screenshot for sharing', or privacy-related image editing. Use this skill even if the user just says 'blur this' with an image file."
---

# Blur Image

Detect and blur sensitive regions in screenshots using AI vision + ImageMagick.

**Announce:** "I'm using the blur-image skill to identify and blur sensitive regions."

**Use when:** Blur, redact, anonymize, hide text in screenshots, prepare images for sharing, remove PII from images.

**Skip when:** Crop, resize, format conversion, color adjustments, or non-privacy image editing.

**Common mistake:** Output goes to `<name>-blurred.webp` by default, not overwriting the original. If the user expects in-place editing, they need `--overwrite`.

## Critical Rules

| Always | Never |
|--------|-------|
| Check `which magick` before running; if missing, provide install instructions and stop | Run blur without user confirmation |
| Show the generated command before executing | Overwrite the original file without `--overwrite` flag |
| Use sigma >= 15 for privacy-sensitive content (low sigma is reversible) | Use sigma < 10 (can be reversed with deblurring algorithms) |
| Save output as WebP with `-quality 85` | Blur decorative/non-sensitive content without asking |
| Add 30-50px padding around detected regions, clamped to image edges | Trust raw AI coordinates without padding (20-50px error margins) |
| Read output image after blurring to verify correct regions were blurred | Skip the verification step |
| Get image pixel dimensions with `magick identify` before generating coordinates | Assume coordinates without checking actual image dimensions |
| Present findings in human-readable terms (categories and descriptions) | Show pixel coordinates, region dimensions, or offsets to the user |

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
2. Run `magick identify -format '%wx%h\n' image.png` — this validates the file exists, format is supported (png, jpg, jpeg, webp, tiff), and returns pixel dimensions in one step. If the command outputs multiple lines, the image is animated (GIF, animated WebP, APNG) — stop and tell the user, because `-region` blur only affects the first frame
3. Report dimensions to confirm the coordinate space — Retina/HiDPI screenshots may be 2x logical resolution

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
4. **Add 30-50px padding** to all sides of each detected region — AI spatial detection has 20-50px error margins. Clamp coordinates so regions don't extend past image edges (X >= 0, Y >= 0, X+W <= image width, Y+H <= image height)
5. Present findings grouped by type — describe what you found in plain language, never pixel coordinates. Users should not see region dimensions, offsets, or pixel positions:
   ```
   I found sensitive content in 3 areas:

   - .env values: 4 secret values after the = signs
     (DATABASE_URL, STRIPE_SECRET_KEY, SENDGRID_API_KEY, JWT_SECRET)
   - Authorization header: Bearer token in the curl command
   - API response: personal data (login, email, name fields)

   Blur all of these? Or tell me which to skip.
   ```
6. If the content has structure (key=value pairs, JSON, config files), ask targeted questions to clarify intent:
   - "Blur just the values after the = signs, or the entire lines?"
   - "The JSON response has login, email, and name — blur all three?"
   - "There's an internal URL in the error output — want that blurred too?"
   Users describe what to blur in human terms. You handle all the geometry internally.
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
  -strip -quality 85 output-blurred.webp
```

**Solid fill (`--solid` flag):**
```bash
magick input.png \
  -region WxH+X+Y -fill black -colorize 100 \
  -region WxH+X+Y -fill black -colorize 100 \
  -strip -quality 85 output-blurred.webp
```

- Default output filename: `<original-name>-blurred.webp` alongside the original (`-strip` removes EXIF metadata)
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

I found sensitive content in 2 areas:

- Config values: an email address and API key after the = signs
- Header section: an Authorization Bearer token

Blur both? Or tell me which to skip.

User: blur all

Running blur...

✓ Done. Saved to screenshot-blurred.webp
I've verified the output — sensitive values are blurred, labels and structure are intact.
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
