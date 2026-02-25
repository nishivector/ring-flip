# RING FLIP — Design Document
**Round 13 | Mode: Initial Design**

---

## DESIGN_DOC
```
game_name: ring-flip
display_name: RING FLIP
```

---

## IDENTITY

**Game name:** RING FLIP
**Tagline:** Every orbit is a countdown.

**What is the player:**
A gravitational anchor — a collapsed star core that has pulled all nearby debris into concentric orbits and cannot escape what it has drawn in. You are stationary. You do not shoot. You do not move. You only flip.

**World feel:**
You are at the center of a dead solar system. Hot debris — fragments of whatever you last destroyed — circles you in concentric rings of cold fire, close enough to see the texture of their velocity. There is no sky, no horizon, no up: only the rings, the dark between them, and the knowledge that they are coming back around.

**Emotional experience:** Precision dread → conducted mastery

**Reference games:**
- **R-Type** — you read the formation before it reaches kill range; anticipation IS the loop; the player's only job is to project forward in time
- **Tony Hawk Pro Skater** — chain kills compound into multipliers; a flip timed perfectly feels exactly like landing a combo you called three seconds early
- **Osmos** — orbital physics as tactile language; the environment is a living clock you must read, not fight

---

## VISUAL SPEC

| Element | Value |
|---|---|
| Background | `#0A0A1A` |
| Primary (player/action) | `#00E5FF` |
| Secondary (enemies) | `#FF4D00` |
| Accent A (chain flash) | `#FFFFFF` |
| Accent B (secondary sparks) | `#FFB300` |
| Bloom | **YES** — strength `1.8`, threshold `0.08` |
| Vignette | **YES** — intensity `0.65`, feathered to screen edge |
| Camera | Top-down, locked. `camera.position.set(0, 0, 18)` looking straight down. **Orthographic projection**, view width = 12 world units. |
| Player silhouette | Radiant cross with inner ring |

### Player Core Visual (exact)
Two concentric circles:
- Inner filled circle: radius `8px`, solid `#00E5FF`
- Outer stroke circle: radius `16px`, stroke width `1.5px`, `#00E5FF` at `0.4` opacity
- Four tick marks at 0°/90°/180°/270° extending `6px` outward from the outer circle's edge, stroke width `1.5px`, `#00E5FF` at `0.6` opacity
- The whole player pulses: inner circle radius breathes between `8px` and `10px` over `1.8s` loop (sine ease)

### Orbit Ring Visual (exact)
Each ring is a full circle drawn as a dashed arc:
- Dash length: `8px`, gap: `4px`
- Stroke width: `1.5px`
- Color: `#1A1A3A` — barely visible, like a printed circuit trace
- Opacity: pulses from `0.3` to `0.6` once per complete revolution of that ring (locked to orbit period)

### Enemy Shape (exact)
A sharp angular wedge — like a broken tooth pointing in the direction of travel:
- Width: `14px`
- Height: `18px`
- Shape: isoceles triangle with the apex pointing forward (direction of orbit)
- Fill: `#FF4D00`
- A thin inner line from base midpoint to apex: `1px`, `#FFFFFF` at `0.3` opacity (suggests "cracked" texture)
- Enemies rotate to always face their direction of travel

### Chain Explosion Visual (exact)
When enemies collide:
- Radial burst of `12` white lines
- Base length: `20–40px` (random per line within range)
- Duration: `180ms`, ease-out
- Color fades from `#FFFFFF` to `#FFB300` over the duration
- Each additional enemy added to the chain: burst radius increases by `+8px`, line count increases by `+4`
- Screen flashes to `#FFFFFF` at `0.15` opacity for `60ms` on any collision, scaling to `0.3` opacity for chain of 5+

---

## SOUND SPEC

**Music:**
Two permanent layers:
1. **Layer 1 (foundation):** Sub-bass sine wave at `55Hz`, amplitude modulated by a `0.25Hz` LFO (slow throb). Volume: 100% of music bus. Constant throughout play.
2. **Layer 2 (rhythm):** 128 BPM hi-hat pattern — closed hat on every 16th note (`31.25ms` intervals), open hat on beat 3 of each bar. Volume: `30%` of music bus by default.

Both layers loop continuously. Music does NOT stop during gameplay events — it underlies everything.

**Music BPM:** 128

**Music responds to:**
- **Combo of 3+ enemies in one chain:** Layer 2 volume jumps from `30%` to `70%` for exactly 2 bars (`3750ms`), then returns to `30%`
- **Flip cooldown active:** Layer 1 LFO rate doubles from `0.25Hz` to `0.5Hz` for the duration of the cooldown (faster anxious throb), returns to `0.25Hz` when cooldown expires
- **Player loses a life:** Both layers drop to `10%` volume for `500ms` (the breath knocked out), then restore normally over `300ms`
- **Wave cleared:** A single `1-bar` musical stab: D minor chord (`D3`, `F3`, `A3`, `C4`), piano + string timbre, `0.8s` duration, played once over the music

**Sound effects (6 — Tone.js implementations):**

1. **FLIP** (enemies reverse direction)
```js
const flip = new Tone.MetalSynth({
  frequency: 200,
  envelope: { attack: 0.001, decay: 0.12, release: 0.3 },
  harmonicity: 5.1,
  modulationIndex: 32,
  resonance: 3200,
  octaves: 1.5
});
flip.triggerAttackRelease("8n");
```
Sharp metallic clank, mid-range, decisive — the sound of orbital direction reversing.

2. **ENEMY COLLIDE — first in chain**
```js
const boom = new Tone.MembraneSynth({
  pitchDecay: 0.15,
  octaves: 10,
  envelope: { attack: 0.001, decay: 0.4, sustain: 0, release: 0.5 }
});
boom.triggerAttackRelease("C1", "8n");
```
Deep chest boom. Felt more than heard.

3. **CHAIN +1 — each additional enemy**
```js
// Same MembraneSynth instance as above.
// Pitch increments by one semitone per chained enemy:
// 1st collision: C1 | 2nd: C#1 | 3rd: D1 | 4th: D#1 | 5th: E1 | ...
// This creates the ascending CLANG cascade — each enemy adds a higher ring
const chainNotes = ["C1","C#1","D1","D#1","E1","F1","F#1","G1"];
boom.triggerAttackRelease(chainNotes[chainIndex], "8n");
```

4. **COOLDOWN EXPIRE** (flip button becomes available)
```js
const ping = new Tone.Synth({
  oscillator: { type: "triangle" },
  envelope: { attack: 0.01, decay: 0.08, sustain: 0, release: 0.1 }
});
ping.triggerAttackRelease("A4", "32n");
```
Soft, high, clean — a watch chime saying "ready."

5. **PLAYER HIT** (enemy reaches player core)
```js
const hit = new Tone.Synth({
  oscillator: { type: "sawtooth" },
  envelope: { attack: 0.001, decay: 0.6, sustain: 0, release: 0.3 }
});
const dist = new Tone.Distortion(0.9).toDestination();
hit.connect(dist);
hit.triggerAttackRelease("G2", "4n");
```
Torn, ugly, wrong. Jarring against the clean orbital music.

6. **WAVE CLEAR**
```js
const reverb = new Tone.Reverb({ roomSize: 0.9, wet: 0.7 }).toDestination();
const poly = new Tone.PolySynth(Tone.Synth).connect(reverb);
poly.triggerAttackRelease(["D3","F3","A3","C4"], "2n");
```
D minor chord. Rings out into the void. Clean and earned.

---

## MECHANIC SPEC

**Core loop:** Read the orbital phases of all rings, find the moment where reversing every enemy's direction sends the maximum number into collision, tap the single flip button, watch the chain detonate.

**Primary input — single flip button:**
- Rendered as a circle, `72px` diameter
- Position: horizontally centered, `40px` from bottom screen edge
- **Default/available state:** `#FF4D00` at `100%` opacity. A pulsing ring expands from `36px` radius to `48px` radius over `1.2s` loop, opacity `0.6 → 0`
- **Cooldown state:** A clockwise progress arc fills the button in `#1A1A3A` as cooldown counts down. `#FF4D00` fill drops to `20%` opacity. A countdown timer renders at button center: remaining seconds rounded up, monospace font, `14px`, `#FFFFFF`
- **pointerdown on button (cooldown = 0):** Instantly reverse the direction of ALL enemies on ALL rings simultaneously. Start flip cooldown timer. Trigger FLIP sound. All enemy wedges rotate `180°` over `80ms` (they visibly spin to face the opposite direction). Chain detection begins.
- **pointerdown anywhere off-button:** No effect. Dead zone prevents accidental flips.
- **pointerup:** No effect.
- **pointermove:** No effect. This is a one-button game.

**Key timing values:**
| Parameter | Value |
|---|---|
| Flip cooldown (levels 1–2) | `4000ms` |
| Flip cooldown (level 3) | `3500ms` |
| Flip cooldown (levels 4–5) | `3000ms` |
| Enemy wedge rotation on flip | `180°` over `80ms` |
| Chain detection radius | Center-to-center distance `< 28px` = collision |
| Orbital spiral-in trigger | Enemy completes `3` full orbits without being destroyed |
| Spiral-in speed | `12px/sec` for first `2s`, then `20px/sec` |
| Player death trigger | Enemy center reaches player center (distance `< 8px`) |
| Post-death freeze | `1500ms` — all motion stops, screen desaturates |
| Post-death respawn | After freeze: spiraling enemies removed, all surviving ring enemies reset to evenly spaced positions on their rings, white flash on player core `200ms`, orbital motion resumes |

**Lives:** `3` lives per run. HUD: three small orbit-ring icons (`12px` diameter circles), top-left corner, `#00E5FF`. Lost lives display at `20%` opacity (ghosted). Lives never replenish mid-game.

**Difficulty curve — exact per level:**

| Level | Rings | Enemies | Ring Radii | Orbit Speeds | Directions | Cooldown |
|---|---|---|---|---|---|---|
| 1 | 1 | 3 | 140px | 28°/sec | CW | 4000ms |
| 2 | 1 | 4 | 140px | 32°/sec | CW | 4000ms |
| 3 | 2 | 4+3 | 120px, 200px | 36°/sec, 22°/sec | CW, CW | 3500ms |
| 4 | 2 | 4+4 | 120px, 200px | 40°/sec, 24°/sec | CW, **CCW** | 3000ms |
| 5 | 3 | 6+6+3 | 110px, 180px, 260px | 45°/sec, 26°/sec, 15°/sec | CW, CCW, CW | 3000ms |

*Level 5 note: Ring 3 is not present at wave start. It enters at `t+8s`: the 3 outer enemies appear on the `260px` orbit ring one at a time over `3 seconds`, each arriving from the screen edge at their starting angular position.*

**Win condition:**
Wave 5 cleared — all `15` enemies destroyed. All orbital motion ceases. A `1-beat` silence. Then WAVE CLEAR chord. "RING CLEARED" fades in. Score displayed. Tap to restart.

**Lose condition:**
All `3` lives lost. Orbital motion freezes. "GRAVITY WINS" fades in. Final score + maximum chain achieved (displayed as e.g. "BEST CHAIN: ×7"). Tap to restart.

**Score:**
- Chain starts at `100pts` for the first enemy destroyed
- Each successive enemy in the same flip's chain: `previous × 2`
  - Chain of 1: `100pts`
  - Chain of 2: `100 + 200 = 300pts`
  - Chain of 3: `100 + 200 + 400 = 700pts`
  - Chain of 4: `1500pts`
  - Chain of 5: `3100pts`
  - Chain of N: sum of `100 × 2^(k)` for k = 0 to N-1
- Wave clear bonus (all enemies dead from final flip): `+500pts`
- Survive wave without losing a life: `+500pts`
- Score renders top-right, monospace font, `20px`, `#FFFFFF`. Points awarded during chains increment visually with a counter animation (`200ms` roll-up).

---

## LEVEL DESIGN

### Level 1 — "THE CLOCK"
**What's new:** Everything. This is the tutorial encoded in the game itself.

**Setup:** `1` ring at `140px` radius. `3` enemies, evenly spaced at `120°` apart. All orbit clockwise at `28°/sec` (one full revolution = `12.86s`). Flip cooldown `4000ms`.

**The teaching moment:** Enemies are spaced so that after ONE flip at ANY moment, two of the three enemies will reach the same arc position within `2 seconds` — a guaranteed collision without requiring timing skill. The game is teaching: *tap = something explodes*. The third enemy then completes the orbit and — depending on when the player flips the second time — either collides with a remaining enemy or spirals in. The threat of a spiral-in after 3 orbits is introduced passively.

**Goal:** Clear all `3` enemies.

---

### Level 2 — "ASYMMETRY"
**What's new:** Enemies cluster unevenly. The player must now actually LOOK before flipping.

**Setup:** `1` ring at `140px` radius. `4` enemies: two clusters of `2` enemies, each cluster spanning `20°` of arc, with `160°` of arc between the two clusters. Orbit clockwise at `32°/sec`. Flip cooldown `4000ms`.

**The teaching moment:** A flip when the clusters are on opposite sides of the ring kills `2` enemies (both in one cluster collide with each other). A flip when both clusters are approaching the same point of the ring kills `4` enemies (both clusters collide simultaneously). The player discovers waiting is rewarded. "Perfect" badge displayed if wave cleared in `2 flips` or fewer — a non-blocking signal of mastery.

**Goal:** Clear all `4` enemies.

---

### Level 3 — "SECOND RING"
**What's new:** Two rings at different speeds. Reading the phase gap between rings.

**Setup:**
- Ring 1: `120px` radius, `4` enemies clustered into two pairs, `40°/sec`, clockwise
- Ring 2: `200px` radius, `3` enemies evenly spaced, `22°/sec`, clockwise
- Flip cooldown `3500ms`

**The teaching moment:** A flip reverses BOTH rings simultaneously. The two rings' different speeds mean their clusters are never in sync by accident — the player must wait for a moment when the inner ring's gap faces the same angular direction as the outer ring's cluster, then flip. Good flips kill `4–5` enemies. Bad flips kill `2`. The player is now a conductor, not a reactor.

**Goal:** Clear all `7` enemies.

---

### Level 4 — "OPPOSITE DAY"
**What's new:** Two rings already orbiting in opposite directions before the first flip.

**Setup:**
- Ring 1: `120px` radius, `4` enemies in two clusters, `40°/sec`, **clockwise**
- Ring 2: `200px` radius, `4` enemies in two clusters, `24°/sec`, **counter-clockwise**
- Flip cooldown `3000ms`

**The teaching moment:** The two rings naturally converge at certain angular positions — enemies from Ring 1 and Ring 2 pass through the same spatial position at regular intervals. A flip at that moment causes cross-ring collisions for the first time. The player discovers that enemies from different rings CAN collide. This is the setup for level 5.

**Goal:** Clear all `8` enemies.

---

### Level 5 — "THE CONVERGENCE"
**What's new:** Three rings. The WOW MOMENT. Everything at once.

**Setup:**
- Ring 1: `110px` radius, `6` enemies (3 clusters of 2), `45°/sec`, clockwise
- Ring 2: `180px` radius, `6` enemies (2 clusters of 3), `26°/sec`, counter-clockwise
- Ring 3: `260px` radius, `3` enemies (evenly spaced), `15°/sec`, clockwise — **enters at `t+8s`**, one enemy per second appearing at its orbit position on the screen edge
- Flip cooldown `3000ms`

**The convergence moment:** Approximately every `8–11` seconds (exact timing depends on starting phases), all three rings' leading clusters align within `30°` of the same angular position. For a `10x` player who has been mentally tracking all three phases, this is the moment. A single flip at convergence sends Ring 1 and Ring 2 into a cross-ring cascade and Ring 3's three enemies into each other. The screen detonates with `15` chain kills: ascending CLANGs from C1 to G1, white burst expanding to fill the ring, amber sparks raining outward, chain score climbing through `3100+ pts`.

**The 1x player** will flip opportunistically and clear the wave in 4–5 flips over 45 seconds.
**The 10x player** waits for convergence, flips once, ends the wave in under 15 seconds with maximum score.

**Goal:** Clear all `15` enemies.

---

## THE MOMENT

Wave 5, approximately `9 seconds` in: all three rings are running, the outer ring of 3 slow enemies has just finished appearing — the player waits, watching the three leading clusters drift toward the `6 o'clock` position on the screen, the flip button pulses orange, the sub-bass throbs faster because the cooldown just expired — they tap — and 12 enemies from two rings instantly reverse into each other in a cascade of 7 ascending CLANGs while the 3 outer enemies reverse and collide simultaneously, filling the screen with white lines and amber sparks the player did not consciously plan but in the next second completely and fully owns.

---

## EMOTIONAL ARC

**First 30 seconds:**
Three glowing orange wedges circle a blue core. A pulsing orange button sits at the bottom of the screen. You tap it out of curiosity. The wedges reverse — you can see them flip. Two slam together. The screen flashes white. A deep boom hits your speakers. Your brain understands: tap = enemies reverse = they crash. You tap again immediately. No text required. No tutorial. The game already taught you everything.

**After 2 minutes:**
You're on level 3 with two rings moving at different speeds. You've stopped watching individual enemies. You're reading the gap between clusters — the dark space between two orange shapes that's either closing or opening. The sub-bass throbs on every revolution. Your thumb is hovering, not tapping. You feel the moment arriving three seconds before it does. You flip. Four enemies die. You exhale.

**Near the win:**
Three concentric rings, alternating directions, enemies at different speeds. The flip button is live. Your eyes are on the lower half of the screen where three clusters are converging. You can feel the shape of the next two seconds. You've been here before in level 4 — you know what this looks like. You wait one more second. Two more seconds. The clusters compress toward the same point. You tap. The screen goes white.

---

## THIS GAME'S IDENTITY IN ONE LINE

**"This is the game where you do nothing for 6 seconds and then detonate everything at once."**

---

## START SCREEN

### Idle Animation (Three.js canvas):
Two orbital rings are already in motion when the start screen appears:
- **Inner ring:** radius `140px`, `3` enemy wedges at `120°` spacing, orbiting clockwise at `10°/sec`
- **Outer ring:** radius `200px`, `3` enemy wedges at `120°` spacing, orbiting counter-clockwise at `7°/sec`
- Both rings' orbit arc lines are rendered at `0.3` opacity, pulsing slowly

Every `6 seconds`, a synthetic flip event fires:
- All wedges reverse direction over `80ms`
- Two inner-ring enemies (pre-positioned to converge) collide: chain explosion fires (`12` white lines, `180ms`, fade to amber)
- `1.5 seconds` later: an outer-ring enemy reaches a position where a remaining inner-ring enemy is orbiting — second collision, smaller burst
- The rings continue orbiting. The demo loop is infinite — perpetual near-misses and detonations in the void, no UI interaction required.

Player core pulses at center throughout.

### SVG Overlay:
```svg
<svg
  width="100%"
  height="100%"
  viewBox="0 0 400 800"
  xmlns="http://www.w3.org/2000/svg"
  style="position:absolute;top:0;left:0;pointer-events:none"
>
  <defs>
    <filter id="title-glow" x="-30%" y="-30%" width="160%" height="160%">
      <feGaussianBlur in="SourceGraphic" stdDeviation="6" result="blur"/>
      <feComposite in="SourceGraphic" in2="blur" operator="over"/>
    </filter>
    <style>
      @keyframes fadeIn {
        from { opacity: 0; transform: translateY(8px); }
        to   { opacity: 1; transform: translateY(0px); }
      }
      @keyframes fadeInSub {
        from { opacity: 0; }
        to   { opacity: 0.85; }
      }
      @keyframes pulse {
        0%, 100% { opacity: 0.5; }
        50%       { opacity: 1.0; }
      }
    </style>
  </defs>

  <!-- Main title -->
  <text
    x="200" y="660"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="52"
    font-weight="bold"
    letter-spacing="0.15em"
    fill="#00E5FF"
    filter="url(#title-glow)"
    opacity="0"
    style="animation: fadeIn 1.2s ease-out 0.4s forwards"
  >RING FLIP</text>

  <!-- Tagline -->
  <text
    x="200" y="698"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="13"
    letter-spacing="0.22em"
    fill="#FF4D00"
    opacity="0"
    style="animation: fadeInSub 0.9s ease-out 1.3s forwards"
  >EVERY ORBIT IS A COUNTDOWN</text>

  <!-- Tap prompt -->
  <text
    x="200" y="756"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="14"
    letter-spacing="0.18em"
    fill="#FFFFFF"
    opacity="0"
    style="animation: pulse 1.6s ease-in-out 2.0s infinite"
  >TAP TO BEGIN</text>
</svg>
```

The Three.js canvas renders behind the SVG. The title floats in the lower third of the screen, leaving the top two-thirds clear for the idle animation to read.

---

## DEAD STATE DEFINITION

**What kills the player:**
An enemy completes `3` full orbits without being destroyed and begins spiraling inward. If the spiraling enemy reaches the player core (center distance `< 8px`), the player loses one life.

**Visual feedback for spiral-in:**
- The moment an enemy begins spiraling, it changes color from `#FF4D00` to `#FFFFFF` and begins leaving a faint trail (`4px` wide, `#FFFFFF` at `0.2` opacity, trailing `200ms`)
- The orbit ring the spiraling enemy was on begins flashing: opacity `0.8` for `200ms`, then `0.15` for `200ms`, repeating until the enemy is destroyed or hits the core
- A thin red indicator line draws from the player core toward the spiraling enemy, visible at `0.4` opacity — a warning vector

**How to kill a spiraling enemy:**
A flip will also reverse the spiraling enemy's direction, snapping it back to a counter-spiral (outward). If it collides with another enemy during the counter-spiral, it is destroyed normally and awards the same points as a chain collision.

**Respawn:**
After life loss — `1500ms` freeze (all motion stops, screen desaturates to `30%` saturation), then: spiraling enemies removed, remaining ring enemies redistribute to evenly-spaced positions on their rings, player core pulses white `200ms`, orbital motion resumes at full speed.

---

*RING FLIP — Design Document | Round 13*
*"Do nothing for 6 seconds. Detonate everything at once."*
