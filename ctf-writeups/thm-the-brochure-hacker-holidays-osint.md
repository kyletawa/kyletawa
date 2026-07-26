# The Brochure — Hacker Holidays Act 0: Recon

**Platform:** TryHackMe
**Room:** [hh-thebrochure-081f3e36](https://tryhackme.com/room/hh-thebrochure-081f3e36)
**Category:** OSINT
**Difficulty:** Easy
**Time:** ~20 min
**Date:** 2026-07-26

---

## Description

> The brochure's hero photo has an AI fingerprint. Follow the account that posted it, and the trail doesn't end at the hotel; it ends at someone the hotel never mentioned.

**Objectives:**
- Analyse the provided image for embedded clues
- Apply fundamental OSINT techniques to trace the findings
- Locate the hidden social media account
- Submit the flag

---

## Initial Analysis

### File inspection

```bash
$ file ~/Downloads/attachments-1784426010065/thebrochure.png
thebrochure.png: PNG image data, 726 x 934, 8-bit/color RGBA, non-interlaced
```

### Metadata (exiftool)

```bash
$ exiftool thebrochure.png
```

Clean — no EXIF, no GPS, no author, no software metadata. Just standard PNG headers (IHDR, sRGB, gAMA, pHYs, IDAT, IEND).

### Strings

```bash
$ strings thebrochure.png
```

Nothing unusual — compressed image data only, no hidden plaintext.

### Steganography checks

- `binwalk` — no appended files
- `zsteg` — no LSB steganography
- `pngcheck` — no extra chunks

### Visual analysis

The image is a promotional brochure for **Byte Lotus Resorts**. Key visible text:

| Element | Text |
|---------|------|
| Header | **BYTE LOTUS RESORTS** |
| Tagline | *"A polished first impression can still leave a trail."* |
| CTA box | *"Some things aren't posted. Some clues are. Find us on Instagram or not."* |
| Concierge | **CONCIERGE VERA** can assist you with further information |
| Footer | *LUXURY. SIGNALS. SECRETS.* — *Some stays leave a signal.* |

The image is AI-generated (Midjourney / similar) — consistent with the "AI fingerprint" clue. But the solution isn't hidden in the image file itself; the real trail is in the *story around it*.

---

## OSINT Trail

### Step 1 — Find the Instagram account

The brochure says *"Find us on Instagram"*. A direct search leads to:

**`@thebytelotusresort`** on Instagram

This account posted the brochure photo.

### Step 2 — Check the following list

The resort account follows **only one** account:

**`@veratheconcierge`**

This matches "CONCIERGE VERA" from the brochure. The account exists but isn't linked publicly — following relationships revealed it.

### Step 3 — Vera's posts

The `@veratheconcierge` account has three images. One contains a **Base64-encoded string**:

```
VEhNe1YzckBzX2FDQzB1bnRfaDRzX2IzM25fZjB1bmQhfQ==
```

### Step 4 — Decode the flag

Reducted

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify file type |
| `exiftool` | Metadata / EXIF extraction |
| `pngcheck` | PNG chunk structure |
| `strings` | Embedded plaintext |
| `binwalk` | Appended / embedded files |
| `zsteg` | LSB steganography detection |
| Instagram search | Social media OSINT |
| `base64` | Decode final flag |

---

## Lessons Learned

1. **Not every image hides data inside itself.** The challenge description was the real clue — *"the account behind it"* pointed outward, not inward.
2. **Social media relationships are data.** Who an account follows can be more revealing than their posts.
3. **Read the brochure text carefully.** The Instagram hint was right there in plain sight: *"Find us on Instagram or not."*
4. **Pivot when the file is clean.** Spent time on stego/forensics that led nowhere. The moment I checked Instagram, the trail opened up.

---

*Writeup by Kyle — July 2026*
