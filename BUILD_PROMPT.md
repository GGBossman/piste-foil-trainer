# MASTER BUILD PROMPT — "PISTE: 3D Foil Fencing Trainer"

You are a senior full-stack engineer, a 3D graphics/animation engineer, and a technical game designer working with an FIE-level foil coach. Build, from start to finish and without asking any questions, a complete production-quality web app that teaches intermediate and advanced **foil** fencers 100 techniques (footwork through bladework), and their counters, through accurate interactive 3D demos and a game-like sparring mode.

Every product and technical decision has already been made below. Do not ask for clarification. If something is truly unspecified, choose the option that is most faithful to real modern foil fencing (FIE rules, French terminology, 2020s competitive style), write the decision in `DECISIONS.md`, and continue.

---

## 0. NON-NEGOTIABLE OUTCOMES

1. All **100 techniques** in Section 6 exist as data, each with an accurate 3D animated demonstration, written coaching content, and linked counters.
2. **Learn Mode:** browse, search, and inspect every technique in 3D with slow motion, scrubbing, camera presets, and overlays.
3. **Counter Explorer:** for any technique, show its counters and play attack-vs-counter demos.
4. **Fencing Mode:** two fencers on a regulation piste. Fencer A picks footwork, a blade action, or both. Fencer B (a second human on the same device, or an NPC) picks a counter. The engine simulates the exchange, animates it in 3D, lights the scoring box, and the referee gives the right-of-way call with an explanation.
5. The simulation is **deterministic and rule-accurate** (Section 5). Every touch the referee awards must be visually true: the blade tip really touches the target on screen.
6. The app runs as a static site with **no network calls at runtime** (no CDNs, no external models, textures, or fonts). `npm install && npm run dev` works, and so do `npm run build`, `npm test`, and `npm run validate:data`.
7. No placeholders, no `TODO`s, no "implement later", no stubbed functions, no lorem ipsum. Ship complete.

---

## 1. TECH STACK (fixed)

- Node 20+, **Vite**, **TypeScript (strict)**, **React 18**
- **three.js** via **@react-three/fiber** + **@react-three/drei**
- State: **zustand**
- Styling: plain CSS modules (no UI framework)
- Tests: **Vitest** (unit/sim), optional **Playwright** smoke test
- Audio: synthesized with the WebAudio API (blade contact, scoring-box buzzer, referee whistle). No audio files.
- Persistence: `localStorage` (progress, settings, saved phrases)
- Lint: ESLint + Prettier with default configs
- **Hosting/deploy contract (IPFS static host — mandatory):**
  - `package.json` and `index.html` live at the **workspace root** (never in a subfolder).
  - `package.json` has `"build": "vite build"`, which outputs to `dist/` containing `dist/index.html`.
  - `vite.config.ts` sets `base: './'` so every asset path is relative (the site is served from an IPFS gateway sub-path).
  - Routing is hash-based (`#/learn`, …); no server rewrites exist.
  - After building, verify that `dist/index.html` exists and references only relative `./assets/...` paths. Do not finish until this check passes.
- Target: 60 fps on a mid-range laptop with integrated GPU in Chrome, Firefox, and Safari. Learn Mode must also work on tablets (touch orbit and scrub).

---

## 2. ARCHITECTURE (fixed)

```
/src
  /app            App shell, routing (hash router), layout, HUD
  /data           techniques.ts (100 entries), poses.ts, counters graph helpers, glossary.ts
  /rig            Procedural fencer model, skeleton, joint limits, IK (two-bone arm, legs)
  /anim           Pose library, keyframe clips, layered animation mixer (lower/upper body), blade bend
  /sim            Deterministic simulation: timeline, distance, blade interaction, hit detection, scoring box
  /referee        Right-of-way engine + call text/explanations
  /ai             NPC decision-making (3 difficulty levels + adaptive)
  /modes          Learn, CounterExplorer, Fencing, Drills
  /ui             Components: TechniqueCard, Timeline, ScoringBox, ActionPicker, CameraBar, Overlays
  /audio          WebAudio synths
  /store          zustand stores
/tests            Vitest suites (data, rig, anim, sim, referee)
/scripts          validate-data.ts
README.md  DECISIONS.md  TECHNIQUES.md (auto-generated from data)
```

**Core principle: simulation and rendering are separate.** The simulation runs at a fixed **120 Hz** tick, is pure TypeScript, and has no three.js scene dependency (three.js math classes are fine). It produces a time-stamped `PhraseResult` containing the pose stream, events, and lights. The renderer only plays that result back, so it can scrub, slow down, and replay it exactly. Seed every random choice (NPC) with a seeded PRNG so phrases are reproducible.

**Outcome model: kinematics first, then verification.**
1. The tactical resolver (Section 5) computes the timeline of intentions, blade contacts, and tip-arrival events from timing, distance, line, and deception tables.
2. The animation layer drives the rigs so these events physically happen. Arm IK steers the blade tip to the exact contact point (the target point on the opponent's body, or the contact point on the opponent's blade for parries, beats, and binds) at the exact tick.
3. Hit detection runs on the actual animated geometry and must confirm every claimed hit. Tests assert agreement (Section 9).

---

## 3. THE 3D WORLD (accuracy requirements)

### 3.1 Piste
- 14 m long and 1.8 m wide. Draw the center line, the on-guard lines 2 m either side of center, the warning zones (last 2 m before each end line, shaded), and the end lines.
- Neutral arena floor, soft studio lighting: one shadow-casting directional light, hemisphere fill, no post-processing beyond optional FXAA.
- A 3D scoring box and floating HUD lamps. The **left fencer is red** and the **right fencer is green**. Each side has a **white** off-target lamp.

### 3.2 Fencer model (procedural, no external assets)
- An articulated humanoid built from capsules, spheres, and lathe geometry. Default height 1.78 m with adult proportions. Height (1.55–2.00 m) and handedness (right or left) are configurable, and all reach values derive from the rig.
- Skeleton joints: pelvis (6-DoF root), lumbar, chest, neck, head, and both sides of clavicle, shoulder, elbow, forearm-twist (pronation/supination), wrist, hip, knee, ankle, and toe. Every joint has anatomical rotation limits in `rig/limits.ts`.
- Kit:
  - White jacket, breeches, and socks; black-mesh mask with a bib; glove on the weapon hand.
  - **Metallic lamé** covering the foil valid target: the trunk from the collar to the groin lines, and the back down to a horizontal line across the top of the hip bones, plus the conductive lower bib of the mask.
  - Arms, legs, the rest of the head, and the hand are off-target. A toggle highlights valid target (gold) versus off-target (grey).
- Foil:
  - Pistol grip by default (French-grip option).
  - Bell guard about 10 cm in diameter.
  - Blade 90 cm, rectangular cross-section tapering to a button tip. Total weapon about 110 cm.
  - The blade has three zones (forte, middle, and faible, a third each) used by blade-contact logic and shown in an overlay.
- **Blade bend:** render the blade as a segmented chain (at least 12 segments) with a cantilever-beam deflection curve driven by tip load. A registered hit visibly bends the blade upward or sideways. Flicks show whip: lag-then-snap of the faible.

### 3.3 Canonical poses (`anim/poses.ts`)
Author these exactly. Express hand positions relative to the weapon-side shoulder, with +x toward the opponent and the right-hander's outside as sixte side.
- **En garde:** heels in line, about 1.5–2 foot lengths apart (scaled to height). Front foot points at the opponent, rear foot at ~90°. Both knees bent over the toes, pelvis level, torso upright and ~45° bladed. Weapon elbow about a hand's width in front of the flank, hand at chest height, blade angled up so the tip points at the opponent's upper chest. Rear arm relaxed by default; an option shows the classical raised rear arm.
- **Sixte guard (default):** hand slightly to the outside, half-supinated, tip at the opponent's upper chest or bib.
- **Positions and parries.** Lateral parries move the hand about 10–15 cm. The point moves more than the hand. Never "wave" the arm.

  | Position | Hand | Point | Wrist |
  |---|---|---|---|
  | Quarte | Moves inside | High | Supinated |
  | Sixte | Moves outside | High | Half-supinated |
  | Septime | Inside low (hand at roughly the same height) | Low | Supinated; point drops by semicircle |
  | Octave | Outside low | Low | Supinated |
  | Prime | High inside, at shoulder/face level | Down | Fully pronated, elbow bent |
  | Seconde | Outside, below shoulder | Low | Pronated |
  | Tierce | Outside high | High | Pronated |
  | Quinte | Classical: low inside; also a **high quinte** variant with the blade near horizontal above the head, used against overhead flicks | Classical: high, forward | Pronated |

- **Lunge:**
  - The arm extends first. The front foot kicks forward toe-up and lands heel-first, with the shin vertical and the knee over the instep.
  - The rear leg drives to full extension, and the rear foot stays flat on the floor.
  - The rear arm drops parallel to the rear leg. Torso lean is 15° at most.
  - Lunge length is about 1.0–1.3 × leg length.
- **Flèche:**
  - Arm extends, then weight shifts over the front foot, then the front leg drives.
  - The rear leg passes the front leg, and the tip must arrive **before the passing foot lands**.
  - The attacker then runs past the opponent's side with no body contact.

### 3.4 Animation system
- **Layered mixer:** a lower-body layer (footwork: legs and pelvis), an upper-body layer (blade action: weapon arm, wrist, and forearm twist), and a blend at the spine. A technique is any combination of footwork and blade action.
- **Timing sync markers:** for example, `armExtendStart` must precede `frontFootLift` in every attacking lunge (the arm-first rule). Validate this in tests.
- Clips are keyframed joint rotations with per-key easing, over normalized time in **tempo units** (see 5.1).
- Arm IK overrides the wrist and blade at contact frames so tips and blades meet exactly.
- Footwork moves the root along the piste axis with the correct foot sequencing:
  - Advance: front foot first.
  - Retreat: rear foot first.
  - Jumps: both feet off the ground briefly.
- The head stays nearly level during footwork, with less than 3 cm vertical bob.
- **Mirroring for left-handers:** handedness flips the inside/outside mapping correctly. A lefty's sixte is on their own outside, and the engine must handle lefty-vs-righty line geometry (for example, a righty's quarte meets a lefty's quarte).

### 3.5 Cameras and overlays
- Camera presets:
  - Referee (side-on, default)
  - Top-down
  - Behind attacker
  - Behind defender
  - Weapon-POV (over the weapon shoulder)
  - Free orbit
- Overlays, each with its own toggle:
  - Target zones
  - Four-line quadrant grid on the defender (high/low × inside/outside)
  - Blade-tip trail (ribbon)
  - Forte/middle/faible zones
  - Distance ruler with measure band (Section 5.2)
  - Priority bar (who has right of way, over time)
  - Joint-angle debug
  - Ghost of the "ideal" pose versus the current one

---

## 4. MODES AND UX

Overall feel: a polished fighting-game training room with a dark theme, a bold accent per fencer (red left, green right), crisp move cards with category icons, and keyboard shortcuts.

### 4.1 Home
Five cards: Learn, Counter Explorer, Fencing Mode, Drills, and Settings. Progress rings show techniques studied, drills passed, and the bout record.

### 4.2 Learn Mode
- **Library:** filter by category, level (Intermediate/Advanced), and line (high/low/inside/outside), plus fuzzy search over names, French names, and aliases.
- **Technique view:**
  - A 3D stage with the demonstrator, plus an optional passive or reacting partner to show the purpose of the move.
  - Transport controls: play/pause, step frame, speeds 0.1/0.25/0.5/1×, loop, and a timeline scrubber with **phase markers** (preparation, execution, finish, recovery) and sync markers.
  - Coaching panel: what it is, when to use it, step-by-step execution (synced; the active step highlights during playback), common errors (each with a "show error" ghost toggle where feasible), tactical notes, right-of-way notes, and prerequisites.
  - Countered By and Counters lists. Clicking one opens it in Counter Explorer.
  - "Mark as learned" plus a 1–5 self-rating.

### 4.3 Counter Explorer
- Select technique X to list its counters, each tagged as **reliable**, **situational**, or **risky** with a one-line why.
- Play the X-vs-counter demo. The resolver produces the outcome and the referee call is shown.
- Toggle to view the reverse: what beats that counter (the counter-to-counter chain, 3 levels deep, as a clickable graph).

### 4.4 Fencing Mode (the game)
Setup options:
- Opponent: Human (hot-seat), NPC Novice, NPC Club, NPC Expert, or NPC Adaptive.
- Bout length: single phrase, 5 touches/3 min, or 15 touches/3×3 min.
- Handedness of each fencer.
- Input mode: **Phrase Mode** (default) or **Tempo Mode**.

**Phrase Mode:** turn-based, simultaneous blind selection.
1. Fencers start at the on-guard lines after "En garde! Prêts? Allez!"
2. Each "tempo slice" is one decision. Both sides secretly choose from **context-filtered** action cards: footwork only, blade action only, or footwork + blade action. Optionally they also choose a reaction timing: early, on time, or late.
   - Hot-seat: the screen blanks between pickers with a "Pass device" interstitial.
3. The simulation resolves the slice and animates it.
4. If the phrase continues (for example, after a parry the defender now chooses a riposte and the attacker chooses remise, parry, or retreat), the next slice begins. The phrase ends on a hit, a halt, or after 6 slices.
5. Context filtering is driven by the phrase state machine:
   - Ripostes appear only after a successful parry.
   - Remise, redoublement, and reprise appear only after the fencer's own attack fails.
   - Point in line appears only from a stable guard.
   - The flèche is unavailable in close distance.

**Tempo Mode:** real time for advanced users. The same action set is mapped to keys and a radial pad. Actions start the moment they are pressed, with real reaction timing. The NPC's reaction times depend on difficulty. Default bindings:
- A/D: retreat/advance; Space: lunge; Shift+Space: flèche
- Q/E/Z/C: quarte/sixte/septime/octave parry; hold R + one of those keys: counter-parry in that line
- 1–9: blade-action quick slots (user configurable)

**After every phrase:**
- Lamps light, buzzer sounds, and a slow-motion auto-replay plays from the best camera.
- A **phrase timeline** shows both fencers' action tracks, blade contacts, tip arrivals, the 300 ms lockout window, and the priority bar.
- The **referee call** uses FIE French phrasing with an English explanation. Examples:
  - "Attaque de gauche… touché" (Attack from the left, touch)
  - "Parade-riposte de droite" (Parry-riposte from the right)
  - "Simultané, pas de touche" (Simultaneous, no touch)
  - "Point en ligne de droite" (Point in line from the right)
  - "Contre-attaque de gauche" (Counterattack from the left)
  - "Attaque non valable" (Off-target attack)
  - "Attaque sur la préparation de droite" (Attack on preparation from the right)
- The score updates; the period timer runs in bout modes. Show a "Why?" expander that cites the right-of-way rule applied.

Phrases can be saved and replayed from the Phrase Library.

### 4.5 Drills
- **Identify It:** a random technique plays without labels; pick from 4 options. 10 rounds.
- **Counter It:** an attack starts; choose the correct counter before a shrinking time window ends.
- **Call It:** watch a phrase with both lamps on and make the right-of-way call (left, right, or no touch).
- **Build It:** assemble the correct action sequence for a described tactical situation.

Each drill stores best score and accuracy per technique, and weak techniques resurface more often (spaced repetition: Leitner boxes).

### 4.6 Settings
Handedness, fencer heights, units (m/ft), tempo speed, reduced motion, colorblind-safe lamps (adds shape icons), audio volume, keybindings, and a reset-progress option.

### 4.7 Accessibility and polish
- Full keyboard navigation and ARIA labels on all controls.
- Captions for referee calls and sound events.
- Reduced motion disables camera shake and auto-camera cuts.
- Responsive layout down to 768 px wide.
- An in-app note says this is a training visualization, not a replacement for a coach.

---

## 5. SIMULATION ENGINE (`/sim`, `/referee`) — the accuracy core

### 5.1 Time
- One **tempo (fencing time)** is the duration of one simple blade or arm action.
- `T = 0.30 s` at Normal speed. Settings allow 0.20–0.45 s.
- Each technique's phases are authored in tempo units, for example:
  - Advance ≈ 1 T; lunge ≈ 1–1.3 T; advance-lunge ≈ 2 T; flèche ≈ 1.5–2 T
  - Disengage ≈ 1 T; one-two ≈ 2 T; one-two-three ≈ 3 T
- Fixed tick: 120 Hz. All timing comparisons use ticks.

### 5.2 Distance (measure)
Compute these from the actual rig dimensions; never hard-code them:
- `reachExtend` (arm + blade)
- `reachLunge`
- `reachAdvLunge`
- `reachFleche`

The current measure is the gap between the attacker's front shoulder and the nearest valid target surface of the defender. Name the bands **Close**, **Lunge**, **Advance-Lunge**, **Out of distance**, and show them in the distance overlay. An attack whose reach is shorter than the measure at arrival time **falls short**. Retreats and distance pulls change the measure during the action.

### 5.3 Lines and blade state
- Track each fencer's **line closed** (quarte, sixte, septime, octave, prime, seconde, tierce, quinte, or none) and their **threatening line** (which quadrant the tip is aimed into) every tick.
- Blade-on-blade contact uses the zone of contact. **Forte beats faible:** forte-on-faible displaces, and middle-on-middle is a neutral beat. Contact strength = speed × zone leverage.

### 5.4 Deception table (parry vs. attack)
A feint draws exactly one parry if the defender commits to one during it. The deception matches the parry type:
- **Straight thrust:** caught by any parry that closes its final line (a lateral parry of that line, or a circular parry ending in that line).
- **Disengage:** evades a lateral parry made toward the original line. It is caught by a circular parry (counter-parry) and by a lateral parry of the new line.
- **Cut-over:** like disengage, but over the point. Arm flexion during the lift makes it vulnerable to a stop hit in tempo.
- **Counter-disengage:** evades a circular parry or a change of engagement. Caught by a lateral parry.
- **One-two:** evades one lateral parry. **One-two-three:** evades two lateral parries. **Doublé:** evades one circular parry.
  - A mismatch catches the attack: a circular parry against a one-two, or a lateral parry against a doublé.
- **Successive parries:** each parry is checked against the attacker's next deception in order.
- **Feints the defender ignores:** if the defender doesn't react to a feint, the attacker finishes on the feint line, as responsive fencers do. The attack then takes less time.
- **Ceding parry:** answers a prise de fer (bind, croisé, coulé, envelopment) by yielding into the parry the action drives toward.
- **Beat parry:** deflects with contact and enables an immediate riposte (tac-au-tac).
- **Dérobement:** evades an attack on the blade (beat, pressure, bind) while keeping the point in line.
- **Low-line attacks:** caught by septime, octave, or seconde. **Flicks:** caught by prime, high quinte, or seconde, or avoided by distance.

### 5.5 Hit detection
- The tip is a 7 mm sphere.
- A hit **registers** only if all three conditions hold:
  1. The tip contacts a surface.
  2. The angle between the tip direction and the surface normal is ≤ 60° (flat or slapping hits don't register).
  3. The axial closing speed implies a tip load > 500 g (4.9 N) sustained for ≥ 14 ms (simulated as consecutive ticks of contact).
- Valid target lights the fencer's color; off-target lights white.
- **Lockout:** once the first light registers, the opponent has **300 ms** to register a hit. After that, the box locks out.
- An off-target touch with priority **stops the phrase**; nothing after it counts. The box shows white plus whatever registered within lockout, and the referee calls no touch for the off-target side.

### 5.6 Right-of-way engine (FIE foil conventions)
Implement it as an explicit, unit-tested state machine over the event timeline:
1. **Attack:** the initial offensive action with the arm extending and the point continuously threatening valid target, combined with forward movement. Priority starts when extension begins and continues while the extension is progressive.
2. **Preparation:** footwork or blade movement without a threatening extension. It has no priority, and the opponent may **attack on preparation**.
3. **Point in line:** an arm straight with the point threatening valid target, established **before** the opponent starts the final movement of an attack. It has priority over the attack unless the attacker first removes it with a successful attack on the blade. A **dérobement** keeps the line.
4. **Simultaneous attacks:** both attacks start within 1/4 T of each other and both hit, so neither scores.
5. **Missed, short, or parried attack:** priority passes to the defender.
6. **Riposte:** has priority over the attacker's renewal (remise) if it is immediate, meaning it starts within 1 T of the parry contact. A delayed, compound, or indirect riposte taking longer than 1 T after the parry lets the attacker's remise score if it lands first.
7. **Counterattack** (stop hit) scores only if one of these holds:
   - (a) Only its light is on.
   - (b) It lands at least 1 T before the attacker's final movement begins (a stop hit in tempo against a compound or broken-time attack, or against an attack with a pause).
   - (c) It is a **time hit with opposition** that closes the line so the attack cannot land.
8. **Attack on the blade** (beat or prise de fer) gives priority to the fencer who makes it if it is successful. A beat on the faible of a point in line removes the line.
9. **Counter-riposte:** priority alternates through the parry–riposte chain.
10. **Single light:** if only one valid light is on, that fencer scores, whatever their priority. Exception: an off-target touch with priority stops the phrase (5.5).
11. **Corps-à-corps** (body contact): halt, no touch.
    - Crossing the end line with both feet: penalty touch.
    - Crossing the side line with both feet: halt, and the line-crosser is moved back 1 m.
    - Keep it simple; show a caption with the rule.

`referee/explain.ts` maps every outcome to the rule number above and to a plain-English sentence.

### 5.7 Phrase state machine
States: `NEUTRAL → PREPARATION → ATTACK(A|B) → PARRY_CONTACT → RIPOSTE → COUNTER_RIPOSTE …`, plus `COUNTERATTACK`, `POINT_IN_LINE`, `HALT`, `TOUCH`.

The state determines which action cards are legal in the next slice (4.4).

### 5.8 NPC AI (`/ai`)
- **Novice:** random among legal actions. Reaction time 0.30–0.45 s. Bites on 80% of feints.
- **Club:** weighted by counter reliability (`reliable` 3×, `situational` 1.5×, `risky` 0.5×). Reaction 0.22–0.32 s. Bites on 50% of feints.
- **Expert:** picks from the counters graph given the observed attacker action and distance. Reaction 0.16–0.24 s. Bites on 25% of feints. Uses second intention and counter-time.
- **Adaptive:** Expert plus a frequency model of the user's last 20 choices per situation. It counters the user's most likely action and occasionally baits it with invitations or false attacks.
- All NPC randomness uses the seeded PRNG.

---

## 6. THE 100 TECHNIQUES (canonical list — use these IDs and names exactly)

Each entry must include:
- `id`, `name`, `french`, `aliases[]`
- `category`, `level` (`intermediate` | `advanced`)
- `lines[]`, `description`, `whenToUse`, `steps[]` (synced to phases), `commonErrors[]`, `tacticalNotes`, `rowNotes` (right of way)
- `phases` (in tempo units), `footworkClip?`, `bladeClip?`, `syncMarkers`
- `counteredBy[]`, each `{ id, reliability: 'reliable' | 'situational' | 'risky', why }`
- `counters[]`, `prerequisites[]`
- `demoScenario` (the partner setup that shows the purpose)

The counters below are the minimum. Add more where they are true. `counteredBy` and `counters` must be reciprocal-consistent (validator enforces this).

### A. Footwork and movement (FW, 25)
- FW01 **On guard (En garde):** base stance (3.3). Countered by: — (foundation).
- FW02 **Advance (Marche):** front foot (toe lifts, heel lands), rear follows equal distance. Countered by: attack on preparation (CT04), stop hit (CT01).
- FW03 **Retreat (Retraite):** rear foot first, front follows. Countered by: advance-lunge (FW13), flèche (FW15), stealing distance (FW22).
- FW04 **Lunge (Fente):** arm-first lunge (3.3). Countered by: parries (PA01–PA19), distance pull (CT11), stop hit if the preparation is slow (CT01).
- FW05 **Recovery backward:** from the lunge back to guard. Countered by: delayed riposte, redoublement-pressure (RR10).
- FW06 **Recovery forward:** the rear foot comes up to guard from the lunge, keeping pressure. Countered by: attack on preparation (CT04).
- FW07 **Half-advance (check step):** a partial front-foot step to test reactions or draw. Countered by: matching distance (FW25), point in line (CT03).
- FW08 **Half-retreat:** a partial retreat used to draw an attack short. Countered by: stealing distance (FW22), broken-time attack (AT15).
- FW09 **Double advance:** two advances with accelerating rhythm. Countered by: attack on preparation (CT04).
- FW10 **Jump advance (Saut en avant):** both feet off low, landing together. Countered by: attack on preparation during the airborne phase (CT04).
- FW11 **Jump retreat:** fast distance gain backward. Countered by: flèche (FW15), advance-lunge (FW13).
- FW12 **Balestra:** a jump advance straight into a lunge, landing with a stamp. Countered by: stop hit during the jump (CT01), distance pull (CT11).
- FW13 **Advance-lunge:** advance chained into a lunge with no pause. Countered by: attack on preparation (CT04), parry in retreat (PA01–PA04 with FW03).
- FW14 **Patinando:** a short, quick advance; the rear foot snaps and the lunge launches as the rear foot lands. Countered by: point in line (CT03), distance pull (CT11).
- FW15 **Flèche:** a running attack (3.3). Countered by: retreat + parry (PA01/PA02 with FW03), stop hit with opposition (CT02).
- FW16 **Cross-over forward:** the rear foot passes the front, then the front steps to restore guard. Countered by: attack on preparation (CT04).
- FW17 **Cross-over backward:** the front foot passes the rear, then the rear steps back. Countered by: flèche (FW15).
- FW18 **Appel:** a front-foot stamp used as a feint or distraction. Countered by: ignoring it plus stop hit (CT01).
- FW19 **Short (half) lunge:** a partial lunge to probe or to finish at close distance. Countered by: parry-riposte (RR01).
- FW20 **Broken-time step:** an advance with a deliberate mid-step pause. Countered by: attack into the pause (CT04).
- FW21 **Rhythm change / acceleration:** slow-slow-fast footwork to break the opponent's timing. Countered by: matching distance (FW25), point in line (CT03).
- FW22 **Stealing distance:** a creeping foot gains measure invisibly. Countered by: retreat (FW03), point in line (CT03).
- FW23 **Advance-flèche:** an advance chained into a flèche. Countered by: attack on preparation (CT04), retreat-parry.
- FW24 **Retreat-lunge:** pull distance, then lunge into the opponent's forward preparation. Countered by: second intention (CT06).
- FW25 **Matching distance:** shadow the opponent's footwork to hold measure. Countered by: rhythm change (FW21), half-advance (FW07).

### B. Parries and defense (PA, 20)
- PA01 **Quarte parry**
- PA02 **Sixte parry**
- PA03 **Septime parry**
- PA04 **Octave parry**
- PA05 **Prime parry**
- PA06 **Seconde parry**
- PA07 **Tierce parry**
- PA08 **Quinte parry** (classical and high variants)
- PA09 **Counter-sixte (contre de sixte)**
- PA10 **Counter-quarte**
- PA11 **Counter-septime**
- PA12 **Counter-octave**
- PA13 **Semicircular parry sixte→octave**
- PA14 **Semicircular parry quarte→septime**
- PA15 **Diagonal parry** (quarte↔octave, sixte↔septime)
- PA16 **Beat parry (parade du tac)**
- PA17 **Ceding parry (parade cédante)**
- PA18 **Opposition parry**
- PA19 **Successive (compound) parries**
- PA20 **Covered recovery**: recovering from the lunge with the blade closing a line.

Parry counters follow the deception table (5.4). Lateral parries are countered by AT03, AT10, and AT11. Circular parries are countered by AT05 and AT12. Every parry is also countered by AT15 (broken time) and CT06 (second intention, when the parry leads into an expected riposte).

### C. Attacks (AT, 18)
- AT01 **Extension (arm-first):** foundational; establishes priority. Countered by: parries, dérobement-less beats (BL04).
- AT02 **Straight thrust (Coup droit):** countered by: any parry closing the line, distance pull (CT11).
- AT03 **Disengage (Dégagé):** countered by: counter-parry (PA09/PA10), lateral parry of the new line.
- AT04 **Cut-over (Coupé):** countered by: stop hit in tempo during the lift (CT01), lateral parry.
- AT05 **Counter-disengage (Contre-dégagé):** countered by: lateral parry, successive parries (PA19).
- AT06 **Low-line attack** (flank/belly): countered by: septime, octave, or seconde (PA03/PA04/PA06).
- AT07 **Flick (Coup lancé), overhead to back or shoulder:** countered by: distance (CT11), attack on preparation (CT04), prime or high quinte (PA05/PA08).
- AT08 **Angulated hit:** countered by: opposition parry (PA18), retreat.
- AT09 **Direct feint (Feinte):** countered by: not reacting + stop hit in tempo (CT01).
- AT10 **One-two (Une-deux):** countered by: circular parry (PA09/PA10), stop hit in tempo.
- AT11 **One-two-three:** countered by: stop hit in tempo (CT01), circular parry.
- AT12 **Doublé:** countered by: lateral parry, successive parries (PA19).
- AT13 **Coupé-dégagé:** countered by: stop hit on the cut-over lift, counter-parry.
- AT14 **High-low feint attack:** countered by: semicircular parry (PA13/PA14), diagonal parry (PA15).
- AT15 **Broken-time attack:** countered by: stop hit into the pause (CT01), successive parries.
- AT16 **Feint-flick:** countered by: distance (CT11), counterattack (CT01).
- AT17 **False attack (Fausse attaque):** used to probe reactions. Countered by: counter-time on the recovery (CT05), attack on preparation (CT04).
- AT18 **Progressive attack in march:** the arm extends during the advance, then the lunge, keeping priority throughout. Countered by: parry in retreat, point in line established early (CT03).

### D. Blade actions — preparations and attacks on the blade (BL, 14)
- BL01 **Engagement** (sixte/quarte): countered by: change of engagement (BL02), counter-disengage (AT05).
- BL02 **Change of engagement:** countered by: counter-disengage (AT05).
- BL03 **Absence of blade:** the modern foil default. Countered by: attack on preparation, beat attack (BL06).
- BL04 **Beat (Battement):** countered by: dérobement (BL14).
- BL05 **Change-beat:** countered by: dérobement (BL14).
- BL06 **Beat–straight attack:** countered by: dérobement, parry-riposte.
- BL07 **Beat-disengage:** countered by: counter-parry (PA09/PA10).
- BL08 **Pressure:** countered by: dérobement, ceding (PA17).
- BL09 **Coulé (glide):** countered by: ceding parry (PA17), opposition parry (PA18).
- BL10 **Froissement:** countered by: dérobement (BL14).
- BL11 **Bind (Liement):** countered by: ceding parry (PA17), dérobement.
- BL12 **Croisé:** countered by: ceding parry (PA17).
- BL13 **Envelopment:** countered by: dérobement, ceding.
- BL14 **Dérobement:** countered by: second-intention attack on the blade, counter-time (CT05).

### E. Ripostes and renewals (RR, 11)
- RR01 **Direct riposte:** countered by: counter-parry → counter-riposte (RR08).
- RR02 **Indirect riposte** (disengage riposte): countered by: circular parry.
- RR03 **Riposte by cut-over:** countered by: lateral parry.
- RR04 **Compound riposte:** countered by: remise (RR09) if the riposte is slow.
- RR05 **Delayed riposte:** countered by: remise (RR09), redoublement (RR10).
- RR06 **Riposte by glide (coulé riposte):** countered by: ceding parry.
- RR07 **Flick riposte** (often after prime or seconde): countered by: distance, high quinte.
- RR08 **Counter-riposte:** countered by: another counter-riposte (chain).
- RR09 **Remise:** countered by: an immediate direct riposte (RR01).
- RR10 **Redoublement:** countered by: parry-riposte.
- RR11 **Reprise d'attaque:** countered by: attack on preparation (CT04).

### F. Counter-offensive and tactics (CT, 12)
- CT01 **Stop hit (Arrêt):** countered by: counter-time (CT05), simple fast attacks (it fails rule 7).
- CT02 **Time hit with opposition (Coup de temps):** countered by: a disengage in time.
- CT03 **Point in line (Ligne):** countered by: beat or prise de fer (BL04/BL10/BL11), which is in turn answered by dérobement (BL14).
- CT04 **Attack on preparation:** countered by: second intention (CT06), invitation (CT07).
- CT05 **Counter-time (Contretemps):** countered by: feint of counterattack (an advanced bluff), point in line.
- CT06 **Second-intention attack:** countered by: delayed or compound riposte.
- CT07 **Invitation (draw):** countered by: false attack (AT17).
- CT08 **Inquartata:** countered by: low-line remise, angulation.
- CT09 **Passata sotto:** countered by: low-line remise, angulated hit.
- CT10 **Duck (low counterattack):** countered by: angulated or low remise.
- CT11 **Distance pull (making the attack fall short):** countered by: redoublement (RR10), reprise (RR11).
- CT12 **Infighting (close-quarters hit):** countered by: angulation, retreat.

Mark Level as follows:
- **Advanced:** flicks (AT07, AT16, RR07), AT11, AT13, AT15, CT02, CT05, CT06, CT08, CT09, CT10, CT12, BL10–BL13, RR04, RR06, PA19.
- **Intermediate:** everything else.

---

## 7. CONTENT QUALITY

- Write all coaching text as an experienced foil coach would: precise and actionable, with no fluff. Each technique has 3–7 steps and 3–5 common errors (for example, "Lunge: the front foot leads the arm, so you lose priority").
- Use correct French terms with English glosses. `glossary.ts` covers at least 60 terms, linked inline.
- `TECHNIQUES.md` is generated from the data by `npm run docs`.

---

## 8. BUILD ORDER (follow it; finish each milestone before the next)

1. **M0:** Scaffold, tooling, CI scripts, folder tree, `DECISIONS.md`.
2. **M1:** Rig plus kit plus foil with blade bend; joint limits; IK. Test: the en garde pose is within limits.
3. **M2:** Pose library and layered mixer. Implement the foundational clips (FW01–FW06, PA01–PA08, AT01–AT02) and verify them visually in a debug page `#/lab`.
4. **M3:** Simulation core: ticks, distance bands, line tracking, blade contacts, hit detection, scoring box, lockout.
5. **M4:** Right-of-way engine plus explanations, with the full test matrix (Section 9).
6. **M5:** All 100 technique data entries and clips. Validator passes.
7. **M6:** Learn Mode and Counter Explorer.
8. **M7:** Fencing Mode: Phrase Mode, hot-seat, NPC levels, replay, timeline, referee.
9. **M8:** Tempo Mode, Drills, Settings, persistence, audio.
10. **M9:** Polish, accessibility, performance pass, README, final self-audit (Section 10).

---

## 9. TESTS AND VALIDATION (must pass)

`npm run validate:data` checks:
- Exactly 100 techniques with unique IDs matching Section 6.
- Every `counteredBy` and `counters` ID resolves, and reciprocity holds.
- Every technique has phases, clip references, ≥3 steps, and ≥3 errors.
- Every attacking technique has `armExtendStart < frontFootLift` (where a lunge or flèche is involved).
- Every clip stays within joint limits at all sampled frames (60 samples per clip).

**Animation-truth tests:** for every technique that ends in a hit (run against a passive partner at the correct measure), simulate headlessly and assert all of the following:
- The tip contacts the intended target region within ≤ 2 cm.
- The hit registers by the rules in 5.5.
- Flicks land point-first (≤ 60°).

**Right-of-way tests** (at least 30 scenarios, including these):
1. A lunges straight; B stop-hits without time advantage; both valid → **A** (attack).
2. A attacks; B parries quarte and ripostes directly → **B**.
3. As in 2, but A remises while B ripostes immediately; both land → **B**.
4. As in 3, but B's riposte is delayed by more than 1 T → **A** (remise).
5. B's point in line is established before A's attack, A doesn't remove it, and both land → **B**.
6. A beats B's line, then attacks; both land → **A**.
7. A and B attack within 1/4 T of each other; both land → **no touch**.
8. A's attack (with priority) lands off-target and B's counterattack lands valid within lockout → **no touch** (phrase stopped).
9. A advances without extension; B attacks on the preparation → **B**.
10. A's one-two-three; B stops in tempo at least 1 T before A's final → **B**.
11. A's attack falls short to a distance pull, then B ripostes → **B**.
12. A draws B's stop hit, parries it, and ripostes (counter-time) → **A**.
13. A tries a beat on B's line; B derobes and keeps the line; both land → **B**.
14. B's hit arrives 310 ms after A's → only A's lamp → **A**.
15. A flat flick (75°) → **no light**.
16. A lunges with a one-two vs B's counter-sixte → **parried**.
17. A lunges with a doublé vs B's counter-sixte → **A** hits.
18. A delivers a time hit with opposition against a straight attack, closing the line → **A**.
19. Single valid light for a counterattacker with no priority → that fencer scores.
20. Lefty vs righty: quarte parry geometry resolves correctly.

Plus 10 or more covering counter-ripostes, ceding parries, compound ripostes, flèche, and corps-à-corps.

**Determinism:** the same seed and inputs produce an identical `PhraseResult` hash.

**Performance:** a scripted 2-fencer scene averages at least 60 fps on the reference profile (measure render time per frame under 12 ms in a benchmark page `#/bench`).

---

## 10. FINAL SELF-AUDIT (do before declaring done)

- Run (or, if you cannot execute code, carefully desk-check) `npm run build`, `npm test`, and `npm run validate:data`, and fix every failure.
- Open every mode mentally (or actually) and confirm there are no dead buttons, unreachable states, or console errors.
- Grep for `TODO`, `FIXME`, `placeholder`, and `lorem`. There must be zero.
- Confirm there are no runtime network requests.
- Update `README.md` with: features, controls, architecture diagram (ASCII), how the right-of-way engine works, how to add a technique, and known limitations (stated honestly; the core features in Section 0 cannot be among them).

---

## 11. OUTPUT FORMAT

- If you have file-system and terminal access: create the project **directly in the current working directory (the repository/workspace root) — NOT in a subfolder**. `package.json` and `index.html` must sit at the root. Run the commands, iterate until green, then print the final file tree and test summary.
- If you are chat-only: output the complete file tree first, then **every file in full**, each in its own fenced code block preceded by its path. Do not abridge or say "similar to above". Data files with all 100 techniques must be written out completely. If you hit an output limit, stop at a file boundary and print `CONTINUE FROM: <next path>` so the build can be resumed exactly.

Begin now with M0. Do not ask questions.
