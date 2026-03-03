---
name: blur-image
description: "Use when the user wants to blur, redact, or anonymize parts of an image — screenshots with API keys, emails, PII, customer data, or any sensitive text. Also triggers on 'hide text in screenshot', 'redact image', 'blur sensitive', 'anonymize screenshot', 'prepare screenshot for sharing', or privacy-related image editing. Use this skill even if the user just says 'blur this' with an image file."
---

# Blur Image

Detect and blur sensitive regions in screenshots using AI vision + ImageMagick.

**Announce:** "I'm using the blur-image skill to identify and blur sensitive regions."

**Use when:** Blur, redact, anonymize, hide text in screenshots, prepare images for sharing, remove PII from images.

**Skip when:** Crop, resize, format conversion, color adjustments, or non-privacy image editing.

**Common mistake:** Output goes to `<name>-blurred.webp` by default, not overwriting the original. If the user expects in-place editing, they need to say so.

## How It Works

The skill combines two tools: AI vision finds sensitive text (the part humans miss), and ImageMagick blurs it (the part AI can't do with pixels). The workflow is: check prerequisites → scan the image → ask the user what to blur → run ImageMagick → verify the output by re-reading it → fix any misses.

## Critical Rules

| Do | Why |
|----|-----|
| Check `which magick` before running | ImageMagick 7+ is required; v6 uses different syntax |
| Run `magick identify -format '%wx%h\n'` first | Validates the file and gives you the coordinate space |
| Use sigma >= 20 for blur, >= 30 for terminal screenshots | Sigma below 10 is reversible with deblurring algorithms. High-contrast text (green on black) bleeds through at low sigma. |
| Anchor blur left edge right after the `=` or `:` delimiter | Padding left eats into labels — pad right and vertically instead. The delimiter is your anchor point. |
| Clamp regions to image edges | X >= 0, Y >= 0, X+W <= width, Y+H <= height |
| Present findings in plain language, not coordinates | Users think in "the email in the sidebar," not "240x18+350+92" |
| Ask 1-3 clarifying questions based on what you actually found | People have different sensitivity thresholds — one person's "public info" is another's PII |
| Read the output image back and verify each region | A blur that misses a few characters of an API key defeats the purpose |

## Phase 1: Preflight

1. Run `which magick`. If missing, show install instructions and **stop**:
   - macOS: `brew install imagemagick`
   - Ubuntu/Debian: `sudo apt install imagemagick`
   - Other: https://imagemagick.org/script/download.php
2. Run `magick identify -format '%wx%h\n' image.png` — validates the file, confirms format support (png, jpg, jpeg, webp, tiff), and returns pixel dimensions. If multiple lines appear, the image is animated — stop and tell the user (`-region` blur only affects the first frame)
3. Note the dimensions — Retina/HiDPI screenshots are often 2x logical resolution

## Phase 2: Identify and Discuss

This is the conversational heart of the skill. Scan the image, present what you found, and have a focused discussion about what needs blurring.

### Step 1: Scan the image

Read the image with multimodal vision. Look for anything that could identify a person, compromise a system, or reveal internal infrastructure:

- Credentials: API keys, tokens, passwords, connection strings, `.env` values, bearer tokens
- Personal data: emails, phone numbers, SSNs, names, physical addresses, dates of birth
- Financial: credit card numbers, bank accounts, billing info
- Infrastructure: internal hostnames, private IPs, file paths with usernames, internal URLs
- Business: customer names, account IDs, license keys, proprietary data

This list isn't exhaustive — use judgment. A hostname like `prod-db-3.internal.acme.co` is sensitive even though it's not on any checklist. Err toward flagging things the user can dismiss rather than missing things they'd want blurred.

### Step 2: Present findings

Group what you found by type. Describe in plain language — never show pixel coordinates, region dimensions, or the raw ImageMagick command:

```
I found sensitive content in 3 areas:

- .env values: 4 secret values after the = signs
  (DATABASE_URL, STRIPE_SECRET_KEY, SENDGRID_API_KEY, JWT_SECRET)
- Authorization header: Bearer token in the curl command
- API response: personal data (login, email, name fields)

Blur all of these? Or tell me which to skip.
```

### Step 3: Ask targeted questions

Ask 1-3 questions based on what you actually found. Don't run through a generic questionnaire — focus on the ambiguous cases in *this* image:

- **Structured content:** "Blur just the values after the = signs, or the entire lines?"
- **Partial vs full:** "The database URL has a hostname and password — blur the whole thing, or just the password?"
- **Borderline items:** "There's a person's name in the response — is that sensitive for your use case?"
- **Things you might have missed:** "Is there anything else you'd like blurred that I haven't mentioned?"

If the content is clearly all-sensitive (a screenshot is nothing but credentials), skip the questions and just confirm: "I'll blur all the credential values. Sound good?"

If you found nothing sensitive, say so and offer to blur specific content the user describes.

### User-guided mode

When the user specifies what to blur ("blur the email in the top right"), locate what they described, confirm what you'll blur, and also mention anything else sensitive you noticed: "I'll blur the email. I also see an API key in the terminal — want that blurred too?"

## Phase 3: Blur

Build and run the ImageMagick command. The user already confirmed what to blur — don't ask again.

**Per-line precision for structured content:** When blurring key=value pairs, JSON fields, or config lines, use a separate narrow `-region` per line targeting only the value portion. Size each region to its content — short values get narrow regions, long values get wide ones.

**Anchor the left edge at the delimiter:** The `=` or `:` character is your reference point. Start the blur region right after it (a few pixels past the delimiter character). Add 30-50px of padding to the **right** side and vertically, but do **not** pad left past the delimiter — that clips the label. If you're unsure where the delimiter falls in pixel space, err toward starting the region slightly too far right (a couple value characters peek through, fixable by widening right) rather than too far left (label characters clipped, looks broken). The full label (`JWT_SECRET=`, `"email":`) should always be readable.

**Include value quotes in the blur:** For JSON values like `"email": "jane.doe@acmecorp.com"`, blur the value **including its surrounding quotes** — the visible portion should be `"email": ` and the blurred portion should cover `"jane.doe@acmecorp.com"` (both the opening and closing `"`). Leaving stray quotes visible at blur edges looks sloppy and can hint at value length.

**Command template:**
```bash
magick input.png \
  -region WxH+X+Y -gaussian-blur 0x30 \
  -region WxH+X+Y -gaussian-blur 0x30 \
  -strip -quality 85 output-blurred.webp
```

**Sigma guide:**
| Content type | Sigma | Why |
|-------------|-------|-----|
| Light text on light background | 20 | Low contrast, blurs easily |
| Dark text on white background | 20-25 | Standard documents, web pages |
| Terminal text (green/white on dark) | 30-40 | High contrast bleeds through at lower sigma |
| Bright colored text on black | 40-50 | Very high contrast needs aggressive blur |

If the first attempt doesn't fully obscure the text (you'll check in the next phase), increase sigma or apply the blur twice to the same region.

**Solid fill alternative:** When the user asks for maximum security, or blur isn't working after adjustment, use solid fill — it replaces pixels entirely and can't be reversed:
```bash
-region WxH+X+Y -fill black -colorize 100
```

**Output:** Save as `<original-name>-blurred.webp` alongside the original. `-strip` removes EXIF metadata. Tell the user where the file will be saved before running.

If the user asks to overwrite the original, do so — but confirm first since it's irreversible.

## Phase 4: Verify and Fix

Read the output image back with vision and check each blurred region:

1. **Each region:** Can you still read any characters? Even partial text (a few letters, trailing digits, stray quotes) means the blur failed for that region.
2. **Edges:** Are characters peeking through at the borders? This is the most common failure — the region was slightly too narrow. For JSON values, the blur should cover the value's surrounding quotes too — stray `"` at blur edges means the region needs to extend a few pixels further.
3. **Missed content:** Scan the rest of the image for anything sensitive that wasn't in the original list. Fresh eyes on the blurred version sometimes catch things.
4. **Labels intact:** Confirm the full label is readable — `JWT_SECRET=`, not `JWT_SECR`. If a label is clipped, the blur region started too far left. Shift the region's X position right until the full label (including the `=` or `:`) is visible.

**If something is wrong, fix it:**
- Text still visible → increase sigma or widen the region **to the right**
- Characters at edges → extend the region 20-30px in the leaking direction
- Label clipped → shift the region's X start to the right (never fix label clipping by shrinking — move the region)
- After 2 failed blur attempts on the same region → switch to solid fill for that region
- Re-run and re-verify the fix (don't assume it worked)

**When everything is clean:**
Report what was verified: "Saved to screenshot-blurred.webp. All 4 .env values fully obscured, variable names readable. Bearer token gone. JSON values blurred per-field, keys visible. Want me to adjust anything?"

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

For the .env lines — blur just the values after the = signs,
or the whole lines? And the API response has login, email,
and name — all three?

User: just the values after = and blur all three in the response

Running blur...

✓ Saved to screenshot-blurred.webp

I've checked each region:
- All 4 .env values: fully obscured, full labels readable (DATABASE_URL=, STRIPE_SECRET_KEY=, etc.)
- Bearer token: fully obscured, "Authorization: Bearer" still visible
- JSON login, email, name: each blurred individually (including value quotes), keys and colons visible

Everything looks clean. Want me to adjust anything?
```
