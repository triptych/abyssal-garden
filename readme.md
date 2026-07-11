# Abyssal Garden

An open-ended artificial-life simulation in a single HTML file. No build step, no
dependencies, no framework — one file, vanilla JS, ~1,300 lines.

Creatures with evolvable neural brains swim through a living nutrient field, sense
their surroundings, graze, hunt each other, breed, and diverge into species. There is
no fitness function and no goal state. Whoever manages to eat and reproduce passes on
their genes. Everything you see is a consequence of that.

```
open abyssal-garden.html
```

That's the whole install. It runs offline, from `file://`, on anything with a canvas.

---

## Table of contents

- [What's actually being simulated](#whats-actually-being-simulated)
  - [1. The water — a reaction-diffusion CA](#1-the-water--a-reaction-diffusion-ca)
  - [2. The senses](#2-the-senses)
  - [3. The brain](#3-the-brain)
  - [4. The body](#4-the-body)
  - [5. Selection](#5-selection)
- [Controls](#controls)
- [Parameter reference](#parameter-reference)
- [Things to watch for](#things-to-watch-for)
- [Tuning recipes](#tuning-recipes)
- [Reading the instruments](#reading-the-instruments)
- [Architecture](#architecture)
- [Performance notes](#performance-notes)
- [Extending it](#extending-it)
- [Known limits](#known-limits)
- [Credits](#credits)

---

## What's actually being simulated

Five layers, each feeding the next. None of them are decorative.

```
   ┌─────────────────────────────────────────────────────┐
   │  WATER      Gray-Scott reaction-diffusion field     │
   │             self-organising nutrient blooms         │
   └────────────────────────┬────────────────────────────┘
                            │  grazed by
   ┌────────────────────────▼────────────────────────────┐
   │  SENSES     3 antennae · neighbour range & bearing  │
   │             kinship · own energy · internal clock   │
   └────────────────────────┬────────────────────────────┘
                            │  fed into
   ┌────────────────────────▼────────────────────────────┐
   │  BRAIN      evolvable 10 → 14 → 5 neural network    │
   │             turn · thrust · breed · scent · hunt    │
   └────────────────────────┬────────────────────────────┘
                            │  animates
   ┌────────────────────────▼────────────────────────────┐
   │  BODY       heritable size, sensor geometry, speed, │
   │             oscillator, arm count, hue              │
   └────────────────────────┬────────────────────────────┘
                            │  survives (or doesn't)
   ┌────────────────────────▼────────────────────────────┐
   │  SELECTION  eat → breed → mutate. No fitness fn.    │
   └─────────────────────────────────────────────────────┘
```

### 1. The water — a reaction-diffusion CA

The food is not scattered pellets. It's a **Gray-Scott reaction-diffusion system**, a
continuous cellular automaton running on a toroidal grid at 5px cells:

```
∂u/∂t = Dᵤ∇²u − uv² + f(1 − u)
∂v/∂t = D᷎ᵥ∇²v + uv² − (f + k)v
```

`V` is the nutrient the creatures eat; `U` is the substrate that lets `V` grow.
With `Dᵤ = 0.16`, `Dᵥ = 0.08`, the field self-organises into drifting blooms, spots,
and worm-like structures depending on the feed/kill rates.

This means **the food supply is itself an emergent, living pattern**. Creatures graze
it down, which locally changes the reaction dynamics, which changes where the next
bloom forms. Sparse stochastic re-seeding (the *Fertility* control) keeps the field
from ever being grazed to permanent extinction.

Dead creatures return nutrient to the water at the point of death — a crude but real
decomposition loop.

Because `Bloom feed` and `Bloom decay` are literally the Gray-Scott `f` and `k`
parameters, the panel doubles as a live Gray-Scott pattern explorer. The classic
regimes (mitosis, coral, solitons, chaos) are all in there.

### 2. The senses

Each creature computes 10 inputs per tick:

| # | Input | Meaning |
|---|-------|---------|
| 0 | `fL` | nutrient at the **left** antenna |
| 1 | `fC` | nutrient at the **centre** antenna |
| 2 | `fR` | nutrient at the **right** antenna |
| 3 | `near` | proximity of nearest neighbour (0 = far, 1 = touching) |
| 4 | `∠s` | sin of bearing to that neighbour, relative to own heading |
| 5 | `∠c` | cos of bearing to that neighbour |
| 6 | `kin` | genetic similarity to that neighbour (hue distance) |
| 7 | `E` | own energy, normalised |
| 8 | `osc` | internal oscillator — a heritable-frequency body clock |
| 9 | `ph` | pheromone concentration here (if enabled) |

Antenna **length** and **spread angle** are both genetic. A lineage can evolve long
narrow forward-facing antennae (a hunter's cone) or short wide ones (a grazer's
sweep). You can see this happen.

The kinship input is what makes flocking and kin-avoidance evolvable rather than
hardcoded. Nothing tells a creature to school with its siblings; some lineages
discover it.

### 3. The brain

A fixed-topology feedforward network, `10 → 14 → 5`, with `tanh` hidden units.
Weights and biases are the genome's largest component (10×14 + 14 + 14×5 + 5 = **229
evolvable parameters**).

Five outputs:

| Output | Squash | Effect |
|--------|--------|--------|
| `turn` | tanh | angular velocity |
| `move` | tanh | forward thrust (reverse is possible) |
| `breed` | sigmoid | attempt reproduction if `> 0.55` and energy allows |
| `scent` | sigmoid | deposit pheromone if `> 0.6` |
| `hunt` | sigmoid | attack the nearest neighbour if `> 0.55` |

Note that reproduction is a **behaviour, not a rule**. A creature must decide to breed.
Lineages that never fire the breed neuron simply vanish, so the willingness to
reproduce is itself under selection — as is the timing of it.

### 4. The body

The rest of the genome:

| Gene | Range | What it does |
|------|-------|--------------|
| `hue` | 0–360 | colour — a **near-neutral marker** (see below) |
| `size` | 2.0–6.5 | radius; bigger = eats more per tick, costs more to run |
| `senseDist` | 18–80 | antenna length |
| `senseSpread` | 0.15–1.3 rad | antenna splay |
| `maxSpeed` | 0.6–2.8 | velocity ceiling |
| `freq` | 0.02–0.32 | internal oscillator frequency |
| `arms` | 2–8 | number of radiating cilia (affects only morphology) |

`hue` is the important one. It is inherited with a small random drift and has
**almost no effect on fitness** — it only feeds the `kin` sense. That makes it a
neutral phylogenetic marker, exactly like a molecular clock. Watch a founding colour
drift apart into distinct bands and you are watching **speciation**, rendered
directly. The *Species* counter bins the population into 18 hue buckets and reports
how many are meaningfully populated.

Size and speed carry a real trade-off: larger, faster creatures graze more per tick
but pay quadratically for movement and linearly for existing. There's no "correct"
answer, which is why the population usually splits into fast small skimmers and slow
large tanks.

### 5. Selection

There is no fitness function anywhere in the code. The loop is:

1. Energy starts at `1.05`, caps at `2.6`.
2. Metabolism drains it every tick (a base cost scaled by body size, plus a movement
   cost proportional to **speed squared**).
3. Grazing and predation refill it.
4. If energy exceeds the *Breed threshold* **and** the brain fires `breed`, the
   creature spends 45% of its energy to produce one mutated child, then enters a
   40-tick refractory period.
5. At zero energy it dies and rots into nutrient.

Mutation is per-weight: with probability *Mutation rate*, a gene is perturbed by a
Gaussian of scale *Mutation strength* — and 5% of those are **macro-mutations** that
re-roll the weight entirely, which is what lets the population escape local optima
instead of just polishing one strategy forever.

Predation is deliberately lossy: an attacker recovers only **75%** of the energy it
bites off. Trophic levels cost something, as they do in the real world, which is why
a pure-predator ecosystem always collapses.

---

## Controls

### Mouse / touch

| Action | Result |
|--------|--------|
| **Drag on the water** | paint nutrient — herd a lineage, or build a reef |
| **Click a creature** | open the microscope (live brain, genome, vitals) |
| **Double-click a creature** | zoom to 5× and lock the camera onto it |
| **Scroll / trackpad** | zoom, pinned to the cursor |
| **Pinch** | zoom (touch) |
| **Shift-drag / right-drag / middle-drag** | pan |

### Keyboard

| Key | Action |
|-----|--------|
| `Space` | pause / play |
| `c` | cataclysm — kill 90% |
| `+` / `−` | zoom in / out |
| `0` | fit world |
| `f` | follow the selected specimen |
| `↑ ↓ ← →` | pan |

---

## Parameter reference

Every value below is live — changing it mid-run changes the world mid-run.

### Simulation

| Control | Range | Default | Notes |
|---------|-------|---------|-------|
| Sim speed | 1–5× | 1× | ticks per rendered frame |
| Pause / Step | — | — | `Step` advances exactly one tick |
| Cataclysm | — | — | kills 90% — an extinction event |
| Rain food | — | — | 40 nutrient blooms, scattered |
| Spawn 20 | — | — | injects 20 **random** genomes (fresh blood) |
| Reset world | — | — | new substrate, 90 random founders |

### The Water

| Control | Range | Default | Notes |
|---------|-------|---------|-------|
| Bloom feed | 0.010–0.090 | 0.060 | Gray-Scott `f` |
| Bloom decay | 0.045–0.070 | 0.062 | Gray-Scott `k` |
| Fertility | 0–1 | 0.30 | stochastic nutrient re-seeding |
| Grazing rate | 0.2–3 | 1.0 | how fast a mouth strips a cell |

### Metabolism

| Control | Range | Default | Notes |
|---------|-------|---------|-------|
| Base metabolism | 0.3–2.5 | 1.0 | cost of merely existing |
| Movement cost | 0.2–3 | 1.0 | scales the **v²** term |
| Breed threshold | 1.2–2.4 | 1.70 | energy needed to reproduce (max is 2.6) |
| Population cap | 60–600 | 260 | hard ceiling; births are refused above it |

### Evolution

| Control | Range | Default | Notes |
|---------|-------|---------|-------|
| Mutation rate | 0.01–0.5 | 0.14 | fraction of weights touched per birth |
| Mutation strength | 0.05–1.2 | 0.35 | σ of the Gaussian perturbation |
| Colour drift | 0–24 | 6 | σ of hue inheritance — the molecular clock rate |
| Reseed if dying | on/off | **on** | below 10 individuals, clones survivors up to 24 |

> **Reseed if dying** preserves evolved genomes through a near-extinction instead of
> restarting from random. Turn it **off** if you want to be able to actually lose.

### Interactions

| Control | Range | Default | Notes |
|---------|-------|---------|-------|
| Predation power | 0–1.2 | 0.35 | bite size; `0` disables carnivory entirely |
| Sight range | 0.5–2.2× | 1.0× | global multiplier on evolved antenna length |
| Pheromone trails | on/off | off | enables stigmergy — see below |

### The View

| Control | Range | Default | Notes |
|---------|-------|---------|-------|
| Magnification | 1–12× | 1× | |
| Follow specimen | on/off | off | glides the camera after the selected creature |
| Fit world | — | — | back to 1× |
| Trail persistence | 0.2–0.98 | 0.88 | how long luminous wakes linger |
| Bioluminescence | 0.3–2.2 | 1.0 | glow intensity |
| Water brightness | 0–2 | 1.0 | set to `0` to hide the food and watch pure behaviour |
| Show senses | on/off | off | draws every antenna in the world |
| Fractal bodies | on/off | **on** | radiating cilia; off = plain motes, much faster |
| Diet colour | on/off | off | **overrides hue**: teal = grazer, magenta = predator |

---

## Things to watch for

These are not scripted. They emerge, or they don't, depending on the run.

**Gradient ascent.** The first thing that evolves, usually within a few hundred ticks.
Compare `fL` and `fR`, turn toward the stronger. It's the simplest possible
chemotaxis and it's the same trick *E. coli* uses.

**Edge-grazing.** Blooms have sharp nutrient gradients at their rims. Efficient
lineages learn to ride the rim rather than sit in the middle, because the middle gets
stripped.

**The predator split.** With *Predation power* above ~0.5, the population reliably
bifurcates. Turn on *Diet colour* and watch teal and magenta separate. Then watch the
**population graph oscillate** — those are Lotka–Volterra predator–prey cycles, and
nobody programmed them.

**Kin schooling.** Some lineages evolve positive weights from `kin` and `near` into
`turn`, and start moving in loose shoals. Others evolve the opposite and disperse to
avoid competing with their own siblings. Both are viable.

**Cannibalism avoidance.** A predator lineage that eats its own kin dies out, so
lineages that gate `hunt` on *low* `kin` outcompete those that don't. You can see this
in the brain diagram as a strong inhibitory (coral) edge from `kin` toward `hunt`.

**Punctuated equilibrium.** Hit `c`. The survivors radiate into the empty niches, and
morphology changes fast for a few hundred generations before settling. This is the
whole fossil record in ninety seconds.

**Stigmergy.** Turn on *Pheromone trails*. Creatures can lay scent and smell it. Most
lineages ignore it. Occasionally one evolves the full loop — lay scent when fed,
follow scent when hungry — and builds **ant-like foraging highways** between blooms.
It doesn't always happen. That's the point.

**Sensor investment.** Set *Sight range* to 0.5× and let it run. Antennae get longer
over generations, because the ones that can't find food don't breed. Set it to 2.2×
and they atrophy — sensing isn't free.

---

## Tuning recipes

**A stable garden** *(defaults)* — Slow, pretty, long-lived. Good for watching hue
drift into species over tens of thousands of ticks.

**Predator–prey oscillator**
```
Predation power  0.8
Population cap   400
Breed threshold  1.55
Diet colour      on
```
Watch the population graph. You want the sawtooth.

**Harsh world / fast evolution**
```
Fertility        0.10
Base metabolism  1.8
Mutation rate    0.30
Sim speed        5×
```
Most lineages die. The ones that don't get *good*, quickly.

**Cambrian explosion**
```
Fertility        0.80
Population cap   600
Mutation rate    0.35
Colour drift     18
Fractal bodies   on
```
Chaos, rapid speciation, wild morphologies. Will chug on older hardware.

**Pure Gray-Scott lab**
```
Water brightness 2.0
Population cap   60
Bioluminescence  0.3
```
Then sweep *Bloom feed* and *Bloom decay*. Ignore the creatures; you're here for the
reaction-diffusion.

**Behaviour only**
```
Water brightness 0
Trail persistence 0.96
Show senses      on
```
The food becomes invisible. All you see is the choreography.

---

## Reading the instruments

### The microscope

Click any creature. The diagram is its **actual live neural network**, redrawn every
frame from the real weight matrices.

- **Cyan edges** = excitatory (positive weight)
- **Coral edges** = inhibitory (negative weight)
- **Edge thickness** = magnitude
- **Node brightness** = current activation, this tick

Follow a creature into a bloom and watch `fC` light up, then watch it propagate
through the hidden layer into `move`. That is the animal thinking, and you are
looking directly at it.

`Diet` is a running average of what the creature has actually been eating — it's
observed, not genetic. A creature born to predators that only ever grazes will read
as a grazer.

### The population graph

The sparkline under the stats. Flat is a stable ecosystem. Sawtooth is a
predator–prey cycle. A cliff is an extinction. A slow climb into the cap ceiling means
your world is too generous — raise the metabolism.

### Species count

Population binned into 18 hue buckets; a bucket counts as a species if it holds ≥3
individuals. Crude, but it tracks real lineage divergence because hue is nearly
neutral. It will read `1` at the start of a run and climb.

---

## Architecture

```
abyssal-garden.html
├── <style>            — the whole design system, ~200 lines
├── <canvas id=food>   — substrate, drawn through the camera
├── <canvas id=life>   — trails + creature bodies
├── <canvas id=overlay>— sensors, selection rings (crisp, never trailed)
└── <script>
    ├── camera         — zoom/pan, cursor-pinned zoom, culling
    ├── water          — Gray-Scott step, graze, seed, pheromone field
    ├── brain          — forward pass, mutation
    ├── Creature       — sense → act → draw
    ├── spatial hash   — 48px buckets, O(n) nearest-neighbour
    ├── world          — step, birth/death, reseed, cataclysm
    ├── rendering      — three passes (see below)
    ├── inspector      — live brain diagram
    └── UI             — all bindings
```

### The rendering trick worth knowing about

Glowing trails need a **persistent buffer** (each frame fades the last one slightly
instead of clearing). But a persistent buffer is in a fixed resolution, so naively
zooming it turns everything into mush at 12×.

So the render is split:

1. **Trails accumulate in world space.** Cheap additive dots, faded each tick. Trails
   are diffuse light by nature, so upscaling this buffer reads as *soft luminous wake*
   rather than as blur. Blur is fine here — it's what glow looks like.
2. **Bodies are re-drawn as vectors through the camera transform**, every frame, at
   the current zoom. So anatomy stays razor-sharp at 12×. You can count cilia.
3. **Overlay** is drawn last with `lineWidth = 1 / zoom`, so sensor lines and
   selection rings stay hairline-thin instead of becoming fat ribbons when you zoom.

The world is a **torus** — everything wraps, including the Laplacian, the neighbour
search, and creature movement. There are no walls and no corners to get trapped in.

---

## Performance notes

- **Zooming in makes it faster.** Off-screen creatures are culled from the draw pass.
  At 6× magnification a 90-creature world only draws ~5 of them.
- The reaction-diffusion step is the fixed cost: one pass over `GX × GY` cells
  (~64,000 at 1600×1000) per tick. This is why *Sim speed* is capped at 5×.
- Neighbour lookup is a 48px spatial hash, so population scales roughly linearly, not
  quadratically. 600 creatures is comfortable.
- The radial-gradient glow per creature is the most expensive draw call. If you're
  running hot: turn off **Fractal bodies**, or drop **Bioluminescence**.
- Everything is `Float32Array` where it matters (the two chemical fields, the
  pheromone field, and every weight matrix).

---

## Extending it

The code is commented and sectioned for exactly this. Some obvious next moves:

**Sexual reproduction.** Right now `child()` clones with mutation. Add crossover:
pick two parents in contact, splice their weight arrays at a random point. Watch
whether recombination actually beats asexual lineages (it usually does, but not
always, and the conditions are interesting).

**Speciation barriers.** Refuse to breed when `kin < 0.7`. This turns hue from a
*marker* of speciation into a *cause* of it — reproductive isolation — and the
species count will behave very differently.

**More senses.** `NIN` is a constant; the brain resizes automatically. Add a wall
sense, a corpse sense, a "was I bitten recently" sense. Pain is a rich input.

**Recurrent brains.** Feed `hid[]` from the previous tick back into the input layer.
Suddenly the creatures have short-term memory, and things like ambush behaviour and
path integration become reachable.

**Age and senescence.** There's a hard age cap at 26,000 ticks. Make death
probabilistic and heritable instead, and let the population evolve its own lifespan
against the r/K trade-off.

**Genome export.** Serialise the winning brain to JSON, save it, drop it into a fresh
world. Build a zoo. Run tournaments.

**Multiple substrates.** A second Gray-Scott field with different `f`/`k` = a second
food type. Now you have real dietary niches, and specialists versus generalists.

---

## Known limits

- The population cap is a hard ceiling, not a soft carrying capacity. When you're
  pinned against it, selection effectively stops (births get refused rather than
  competed for). If you want honest selection pressure, keep the population
  comfortably *below* the cap by raising the metabolism instead of lowering the cap.
- Resizing the window re-seeds the water. The creatures survive and are rescaled; the
  chemical field does not.
- `hue` is *nearly* neutral, not perfectly — it feeds the `kin` sense, so under heavy
  predation there's mild selection on colour. This is a feature (it's how kin
  recognition works) but it means the molecular clock isn't perfectly clean.
- Species counting is hue-binning, not a phylogenetic tree. It's an indicator, not a
  measurement.
- One creature only ever perceives its *single* nearest neighbour. Real flocking
  wants a few. This is the first thing I'd change.

---

## Credits

Built with Claude. Techniques standing on the shoulders of:

- **Gray & Scott** (1983) and **Pearson** (1993) — the reaction-diffusion substrate
- **Karl Sims**, *Evolved Virtual Creatures* (1994) — morphology and controller
  co-evolving under no explicit fitness
- **Larry Yaeger**, *PolyWorld* (1994) — neural agents, open-ended selection, the idea
  that you don't need a fitness function at all
- **Craig Reynolds**, *Boids* (1987) — the sensing-and-steering lineage
- **Chris Langton** — Langton's ant, and for coining the term this whole field lives under

No libraries. No build step. One file.

## License

MIT.
