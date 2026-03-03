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
| Confirm what will be blurred before executing (in human terms, not the raw command) | Overwrite the original file without `--overwrite` flag |
| Use sigma >= 15 for privacy-sensitive content (low sigma is reversible) | Use sigma < 10 (can be reversed with deblurring algorithms) |
| Save output as WebP with `-quality 85` | Blur decorative/non-sensitive content without asking |
| Add 30-50px padding around detected regions, clamped to image edges | Trust raw AI coordinates without padding (20-50px error margins) |
| Read output image after blurring and check every sensitive region individually | Skip the verification step |
| Get image pixel dimensions with `magick identify` before generating coordinates | Assume coordinates without checking actual image dimensions |
| Present findings in human-readable terms (categories and descriptions) | Show pixel coordinates, region dimensions, or offsets to the user |
| Ask clarifying questions when intent is ambiguous | Assume what the user wants blurred |

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
2. Scan for all categories of sensitive content. Cast a wide net — it's better to flag something the user can dismiss than to miss something they'd want blurred:
   - **Credentials & secrets:** API keys, tokens, passwords, bearer tokens, OAuth secrets, SSH keys, database connection strings, `.env` values
   - **Personal identifiers:** Email addresses, phone numbers, social security numbers (SSN), national ID numbers, passport numbers, driver's license numbers, date of birth
   - **Financial data:** Credit card numbers, bank account/routing numbers, billing addresses, transaction amounts tied to individuals
   - **Contact & location:** Physical/mailing addresses, personal names (especially full names), usernames, user IDs
   - **Infrastructure:** Internal hostnames, private IP addresses, file paths containing usernames, internal URLs, server names, port numbers on non-standard services
   - **Medical & legal:** Health records, insurance IDs, case numbers, patient identifiers
   - **Business data:** Customer names, account IDs, license keys, internal project codenames, proprietary data
3. For each region, determine the bounding box `(x, y, width, height)` in pixels from **top-left origin** (ImageMagick's coordinate system)
4. **Add 30-50px padding** to all sides of each detected region — AI spatial detection has 20-50px error margins. Clamp coordinates so regions don't extend past image edges (X >= 0, Y >= 0, X+W <= image width, Y+H <= image height)
5. Present findings grouped by type — describe what you found in plain language, never pixel coordinates:
   ```
   I found sensitive content in 4 areas:

   - .env values: 4 secret values after the = signs
     (DATABASE_URL, STRIPE_SECRET_KEY, SENDGRID_API_KEY, JWT_SECRET)
   - Authorization header: Bearer token in the curl command
   - API response: personal data (login, email, name fields)
   - Terminal prompt: your username is visible in the path

   Blur all of these? Or tell me which to skip.
   ```
6. **Ask clarifying questions.** Different users have different sensitivity thresholds. Don't assume — ask when intent is ambiguous:
   - **Structured content:** "Blur just the values after the = signs, or the entire lines?" / "The JSON has login, email, and name — blur all three, or just the email?"
   - **Names and identifiers:** "There's a person's name in the response — is that sensitive for your use case, or is it public info?"
   - **Partial vs. full redaction:** "The database URL has a hostname and password — blur the whole URL, or just the password portion?"
   - **Edge cases:** "There's an internal URL that doesn't contain credentials, but it reveals your infrastructure. Want that blurred?" / "Your terminal prompt shows your username — include that?"
   - **User-specific criteria:** "Is there anything else in this image you'd like blurred that I haven't mentioned? For example, company names, project names, or other context you'd rather not share."
   The user describes what to blur in human terms. You handle all the geometry internally.
7. If no sensitive content found, report that and ask: "I didn't find anything that looks sensitive, but I might be missing context. Is there specific content you'd like blurred? You can describe it in plain language like 'the text in the top right' or 'the company name in the header.'"

### Mode B — User-guided (`--guided` or when user specifies what to blur)

1. User describes what to blur: "blur the email in the top right" or "blur everything below the header"
2. Read the image, locate the described content
3. Describe what you'll blur in plain language and ask user to confirm
4. While looking at the image, also flag anything else that looks sensitive: "I'll blur the email you mentioned. I also noticed an API key in the terminal output — want that blurred too?"

## Phase 3: Generate Command

Build the ImageMagick command with all confirmed regions. The user already confirmed what to blur in Phase 2 — don't ask again. Just run it.

**Per-line regions for structured content:** When blurring key=value pairs, JSON fields, or config lines, use a separate narrow `-region` per line targeting only the value portion. Each line gets its own region sized to its content — short values get narrow regions, long values get wide ones. This keeps labels (variable names, JSON keys) readable while blurring just the secrets. A single large rectangle over the whole block looks sloppy and hides context the user wants visible.

**High-contrast text needs stronger blur:** Green/white text on dark backgrounds (terminal screenshots) bleeds through at low sigma. Use sigma 30-40 for terminal content. If the text is very high contrast (bright green on black), consider a double-pass blur on the same region or sigma 50+.

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
- Tell the user where the output will be saved before running

## Phase 4: Execute and Verify

Verification is the most important phase. A blur that misses even a few characters of an API key defeats the purpose.

1. Run the command
2. Read the output image with multimodal vision and verify each blurred region individually:
   - **Check every region:** Go through each area that was supposed to be blurred. Can you still read any characters? Even partial text (a few letters of an email, the first digits of an SSN) means the blur failed for that region.
   - **Check edges:** Look at the borders of each blur region. Are any characters peeking through at the edges? Quotes, brackets, trailing digits? This is the most common failure mode — the region was slightly too narrow.
   - **Check for missed content:** Scan the rest of the image for anything sensitive that wasn't in the original detection list. Fresh eyes on the blurred version sometimes catch things the first scan missed.
   - **Check labels are intact:** Confirm the non-sensitive parts (variable names, JSON keys, column headers) are still readable and weren't accidentally blurred.
3. If any issue is found:
   - Describe what's wrong in plain language: "The email on line 3 still has the last few characters visible" or "The blur on the Bearer token didn't extend far enough to the right"
   - Fix the coordinates (widen the region, increase padding, boost sigma) and re-run
   - Re-verify the new output — don't assume the fix worked
   - Repeat until clean
4. Report success: "Saved to screenshot-blurred.webp. I've verified each region — all sensitive values are fully obscured, and the labels and structure are intact."
5. Ask: "Want me to blur anything else, or does this look good?"

## Example

```
User: blur the sensitive stuff in screenshot.png

Claude: I'm using the blur-image skill to identify and blur sensitive regions.

Preflight: magick found at /opt/homebrew/bin/magick (v7.1.2)
Image: screenshot.png (1920x1080 PNG)

I found sensitive content in 3 areas:

- .env values: 4 secret values after the = signs
  (DATABASE_URL, STRIPE_SECRET_KEY, SENDGRID_API_KEY, JWT_SECRET)
- Authorization header: Bearer token in the curl command
- API response: personal data (login, email, name fields)

A couple of questions:
- For the .env lines, should I blur just the values after the = signs, or the entire lines?
- The API response has login, email, and name — blur all three?

User: just the values after = and blur all three in the response

Running blur...

✓ Saved to screenshot-blurred.webp

I've checked each region:
- All 4 .env values: fully obscured, variable names still readable
- Bearer token: fully obscured
- JSON login, email, name: each value blurred individually, keys visible

Everything looks clean. Want me to adjust anything?
```

## Red Flags

Stop if you catch yourself:
- Showing pixel coordinates, region dimensions, or the raw `magick` command to the user
- Using sigma < 10 (text may be recoverable with deblurring)
- Overwriting the original without `--overwrite` flag
- Blurring the entire image instead of specific regions
- Not reading the output image to verify the blur
- Reporting success without checking each blurred region individually
- Using coordinates without adding padding
- Skipping `magick identify` to confirm image dimensions
- Not handling Retina/HiDPI resolution differences
- Assuming what the user wants blurred without asking
