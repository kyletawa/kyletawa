# Water Bottle — TryHackMe OSINT

**Platform:** TryHackMe
**Room:** [waterbottle](https://tryhackme.com/room/waterbottle)
**Category:** OSINT
**Difficulty:** Easy
**Time:** ~60 min
**Date:** 2026-07-26

---

## Description

> After returning to my hometown, I needed a water refill from a station I frequently used until 2014, but I've forgotten its name and contact number. I only remember that it is a twelve-digit contact number starting with 63922.
>
> While driving near Boni Avenue, I noticed a new water refilling establishment now stands where the original station used to be. Can you help me find the name and contact number of the original water station?

**Flag format:** `THM{waterstationname_contactnumber}`

**Objective:**
- Identify the original water refilling station at Boni Avenue (as it existed in 2014)
- Find its 12-digit contact number (starting with 63922)
- Submit the flag

---

## Investigation

### Step 1 — Geolocate the area

The challenge mentions **Boni Avenue** — a quick search confirms this is in **Mandaluyong City, Metro Manila, Philippines**.

### Step 2 — Search for water refilling stations

Opened Google Maps and searched for water refilling stations near Boni Avenue. Many results came back. The key was cross-referencing which ones existed **in 2014**, since the challenge states the station was active until then and has since been replaced.

Used **Google Maps Street View history** to inspect each candidate location's 2014 imagery. Two candidates were identified:

| Station | Notes |
|---------|-------|
| Morning Crystal Water Refilling Station | Present in 2014, but no contact starting with 63922 visible on signage |
| Alkafresco Water Refilling Station | Also present, but same issue — no matching number on the board |

Neither had a visible 63922-prefix number, but the search was narrowing.

### Step 3 — Pivot to direct search

Searched for "Aquabest Mandaluyong" based on map findings and found a contact page:

```
https://ph248574-aquabest-mandaluyong-boni.contact.page/
```

**Aquabest - Mandaluyong, Boni** was the original water station. The page listed three contact numbers — one of them matched the 63922 prefix:

```
63 922 872 1288
```

Stripping the country code (63) gives the 12-digit number: **639228721288**

The address confirmed the location:
> Unit D, Villa Maria Apartment, 31 Mayon Street, Boni, Mandaluyong City, Metro Manila, Philippines

### Step 4 — Submit the flag

```
THM{aquabest_639228721288}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Google Maps (Street View history) | Identify the original station via 2014 imagery |
| Web search | Locate the Aquabest contact page |
| `contact.page` directory | Extract the phone number |

---

## Lessons Learned

1. **Historical map data is an OSINT goldmine.** Google Maps Street View history lets you rewind years — essential for finding what *used* to be somewhere.
2. **Not every detail is in the image.** The contact number wasn't on the shop signage. You had to follow the business name to an external directory or contact page.
3. **Country codes matter.** The challenge asks for a 12-digit number starting with 63922, which is the Philippine country code (63) plus the mobile prefix (922). The number format on the site (`63 922 872 1288`) includes the `63`, so the clean 12-digit version is just the digits together.
4. **Corroborate across sources.** Checking both Google Maps history AND a contact page eliminated guesswork between the candidates.

---

*Writeup by Kyle — July 2026*