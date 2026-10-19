# PISTE — 3D Foil Fencing Trainer (build spec)

This repo holds the complete, self-contained build prompt for an interactive 3D foil-fencing trainer (100 techniques, counters, right-of-way simulation, Fencing Mode vs human/NPC).

- **Full spec:** [`BUILD_PROMPT.md`](BUILD_PROMPT.md)
- **Raw URL (for agents):** https://raw.githubusercontent.com/GGBossman/piste-foil-trainer/main/BUILD_PROMPT.md

## Short pointer prompt (for character-limited builders)

```
Build the web app fully specified at:
https://raw.githubusercontent.com/GGBossman/piste-foil-trainer/main/BUILD_PROMPT.md

Step 1: Download that file (curl/wget/fetch). It is the complete, authoritative spec — every decision is already made. Do not ask questions.
Step 2: Before writing code, print the spec's Section 6 technique IDs (FW01–FW25, PA01–PA20, AT01–AT18, BL01–BL14, RR01–RR11, CT01–CT12) to prove you read it in full.
Step 3: Build following its milestones M0–M9; run npm run build, npm test, npm run validate:data until all pass.
Step 4: Deliver the finished project and report the test summary.
If you cannot fetch the URL, stop and say "SPEC UNREACHABLE" — do not improvise the app.
```
