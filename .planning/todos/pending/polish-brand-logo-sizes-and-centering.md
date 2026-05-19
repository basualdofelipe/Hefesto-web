---
title: Polish brand logo sizes and centering across sidebar and login
priority: low
area: ui
source: Phase 12.2 visual verification feedback
created: 2026-04-10
---

## Issues

1. **Login page isotipo not centered** — león renders top-left instead of centered in the CardHeader. The CardHeader has `items-center text-center` but the Image component itself may need `mx-auto` or the parent needs `flex justify-center`.

2. **Sidebar expanded logo too small** — NEMEA text logo at h-12 (48px) is still hard to read because the source image (logo.png) is 1080x1080 with the text occupying only ~30% of the canvas (lots of gray padding). Options:
   - Crop the PNG asset to remove padding (best fix — design task)
   - Increase h-12 to h-16 or h-20 (quick fix but makes header tall)
   - Use a wide/rectangular version of the logo if one exists

3. **Sidebar collapsed isotipo small** — león at size-7 (28px) works but could be bigger. Consider size-8 or size-9.

## Notes

- The images now LOAD correctly (debug fixed the `localPatterns` / `unoptimized` / middleware issues)
- These are purely visual sizing and alignment tweaks
- May also benefit from transparent-background versions of the assets (current logo.png has gray background)
