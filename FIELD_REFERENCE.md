# Yagi Optimiser — Complete Field Reference

Every field in the config file (`.cfg`) and the GUI, what it does, how to
use it, and what to actually type into it. GUI field labels are shown
alongside the matching `.cfg` key wherever they differ.

---

## What is DE?

**DE = Differential Evolution**, the search algorithm underneath
everything. It keeps a whole population of candidate designs at once
(`pop_size` of them) and evolves them together over many rounds
(`generations`), rather than testing one design at a time.

Each round, for every candidate in the population, the algorithm builds a
"trial" design from three *other* candidates already in the population:

```
trial = A + de_F × (B − C)
```

The trial only replaces the original if it scores **at least as well**
(elitist selection) — so the population can never get worse from one
generation to the next, only better or unchanged. That's why the "best
score" printed during a run only ever climbs or stays flat, never drops.

Early on, the population is scattered across the full search range, so
trial designs can be very different from their parents — broad
exploration. As the population converges toward good designs over many
generations, the individuals get closer together, so the same formula
naturally produces smaller and smaller changes — the search narrows
itself down automatically, without any explicit "get more precise" logic.

---

## 1. Frequency & Impedance

### Freq Start (MHz) / Freq Stop (MHz) — `freq_start_mhz` / `freq_stop_mhz`
The band edges the design is optimized across. The search evaluates every
candidate design at several points spanning this range (see **Freq
Points** below) and averages the score — so the result is a compromise
that works reasonably well across the whole band, not just at one
frequency.

### Freq Points — `n_freq_pts`
How many frequency points across that band are checked for every
candidate, every generation. More points = a more thorough check of
in-band performance, but proportionally more computation (runtime scales
directly with this number — see `pop_size`/`generations` below). 3–5 is
typical for a first pass; 7–9 for a more careful final check across a
wide band.

### Target Z0 (ohms) — `target_z0`
Your feed impedance target, e.g. `75.0` for 75-ohm coax, `50.0` for
50-ohm. VSWR is computed against this value.

### Max VSWR — `max_vswr`
The VSWR ceiling. Designs above this get an extra, steeper penalty on top
of the normal continuous VSWR penalty (see `w_swr_penalty` below) — so
this isn't a hard wall, but crossing it costs noticeably more.

---

## 2. Elements & Geometry

### Reflectors / Directors — `n_reflectors` / `n_directors`
Element counts. The driven element is always exactly 1 and isn't counted
in either of these. **The reflector arrangement is derived automatically
from this count** — a single reflector sits at its own boom position; two
or more are always stacked (shared boom position, different heights),
since that's the only arrangement real Yagi designs actually use with
multiple reflectors. There's no separate setting for this — it isn't
something you choose, just something that follows from how many
reflectors you ask for.

### Stack min (frac) / Stack max (frac) — `refl_stack_min_frac` / `refl_stack_max_frac`
Only relevant with more than one reflector. Bounds applied to **every**
pair's spacing independently, as a fraction of the center-frequency
wavelength. If your design needs both a very tight pair and a very wide
one, this range has to cover *both* extremes at once — e.g. `0.03`–`0.65`,
not a narrow band, since the same bounds apply to the innermost and
outermost pair alike. Set this to whatever vertical/lateral spacing is
mechanically practical for your mount, wide enough to include your actual
target values.

### Driven diam (mm) / Element diam (mm) — `driven_diam_mm` / `element_diam_mm`
Tubing diameters. Most real designs use the same diameter for the driven
element as the rest — there's no general rule that it needs to be
thicker. A deliberately thicker driven element (e.g. `25.0` vs `9.53` for
the others) is a specific, purposeful choice some builders make to gain
more bandwidth, not a default assumption to follow unless you have a
reason to.

### Max boom length (m) — `max_boom_length_m`
Hard-ish limit — designs longer than this are penalized (grows with
`w_boom_penalty`, see below).

### Min boom (m) — `min_boom_length_m`
Optional floor, symmetric to the max — designs *shorter* than this are
also penalized. `0` disables it (default). **Without a minimum, nothing
stops the search shrinking the boom** if a shorter, more compact geometry
scores well on gain/F-B — there's no reward for using more of the boom by
itself. If you want a design that actually uses close to the full length
you're allowing (longer booms generally give better pattern control /
lower sidelobes), set this close to the max, e.g. `max_boom_length_m =
4.2` with `min_boom_length_m = 4.0`.

---

## 3. Search Ranges (fraction of wavelength)

All of these bound what the optimizer is *allowed* to try. If the ideal
design needs a value outside these bounds, the search literally cannot
find it — widen the range and re-run. All are expressed as a fraction of
the wavelength at your band's center frequency.

### Refl length min / max — `refl_len_min_frac` / `refl_len_max_frac`
### Driven length min / max — `driv_len_min_frac` / `driv_len_max_frac`
### Director length min / max — `dir_len_min_frac` / `dir_len_max_frac`
Length search range for each element class. Typical starting ranges:
reflectors `0.46`–`0.58`, driven `0.42`–`0.50`, directors `0.36`–`0.46` —
but widen these for an unconventional design (e.g. a K6STI-style design
sometimes needs wider bounds than a textbook Yagi).

### Gap min (all) / Gap max (all) — `gap_min_frac` / `gap_max_frac`
Spacing range between **every** adjacent element pair along the boom —
reflector-to-driven, and every director-to-director gap. Typical range
`0.06`–`0.30`.

### Dir1-ONLY min / Dir1-ONLY max — `dir1_gap_min_frac` / `dir1_gap_max_frac`
A **separate** spacing range just for the driven-to-first-director gap,
defaulting to the same values as the general gap range (no special
treatment). Some designs deliberately use one unusually tight
first-director gap while keeping the rest of the directors
conventionally spaced (K6STI's design uses a ~55mm first-director gap,
for example). **If you widen the general `Gap min (all)` enough to permit
that one tight gap, you also permit every other director gap to go that
tight** — a very different and much worse-behaved kind of geometry
(prone to sidelobe/grating-lobe problems). Keep `Gap min (all)` at a
normal value and loosen only this field instead. For a K6STI-style
55mm-at-89MHz gap: `Dir1-ONLY min = 0.0162`, `Dir1-ONLY max = 0.03` (give
it a little room, don't pin to one exact value).
**Double-check you haven't swapped this with the general Gap min/max —
it's an easy mix-up and silently defeats the whole point if reversed.**

---

## 4. Wire Loss

### Wire material (GUI only) — `wire_resistivity_ohm_m`
The GUI's material dropdown is a quick-select convenience — picking one
fills in the **Resistivity (ohm-m)** field below it (the actual `.cfg`
key is `wire_resistivity_ohm_m`; the material name itself is never
written anywhere). `0` (default) means perfect-conductor wires — no
loss at all, exactly how every version of this tool behaved before this
setting existed, so leaving it at `0` changes nothing.

A positive value applies that resistivity to every wire via NEC's
segment-conductivity loading. This is a real, physically meaningful
setting, not cosmetic — it was verified directly: gain drops by a small,
realistic amount with actual metal resistivities, and by a large, clearly
correctly-directional amount when tested with a deliberately extreme
value, confirming the effect is genuine and scales the right way.

Preset values (ohm-m, at ~room temperature):

| Material | Resistivity | Source |
|---|---|---|
| Zero (lossless) | `0.0` | No loss applied at all |
| Copper | `1.72e-8` | Standard annealed copper reference |
| Aluminum (6061-T6) | `4.0e-8` | Matches EZNEC's own default for this alloy |
| Aluminum (6060-T5) | `3.19e-8` | From its 54% IACS conductivity rating |
| Tin | `1.09e-7` | Standard reference |
| Zinc | `5.90e-8` | Standard reference |
| User Defined | (leave as-is) | Type your own value — for any other alloy or a lab-measured figure |

**The two aluminum entries are specific alloys, not pure aluminum** —
alloying elements measurably raise resistivity above pure aluminum's
figure, and different alloys vary meaningfully from each other (6061-T6
vs. 6060-T5 differ by about 25%). If you're using different tubing,
"User Defined" with your alloy's actual rated resistivity (often quoted
as %IACS conductivity — convert via `resistivity = 1.7241e-8 / (IACS%
÷ 100)`) will be more accurate than picking the closest-sounding preset.

## 5. Search Effort

### Population size — `pop_size`
How many candidate designs are evaluated in parallel each generation.
Must be at least 4 (the search needs 4 distinct candidates per round to
build a trial design; smaller values are automatically bumped up to 4
with a warning). Larger populations explore more broadly per generation
but cost proportionally more time. 20–30 for quick experiments, 60 for a
thorough run.

### Generations
How many rounds of refinement. Total work scales as `pop_size ×
(generations + 1) × n_freq_pts` NEC evaluations — this is what drives
runtime (see the "How long it takes" section of the README/USER_GUIDE).
DE typically converges (stops meaningfully improving) well before large
generation counts — watch the printed "best score" trend rather than
assuming a bigger number here is always better. If the score has been
flat for hundreds of generations, more won't help.

### DE F
The mutation scaling factor in `trial = A + F × (B − C)` (see "What is
DE?" above) — literally, how big a step each trial design takes relative
to the difference between two other candidates currently in the
population. Default `0.6`. This is the main lever for "aggressive vs.
subtle":

- **Higher** (`0.8`–`1.0`): big jumps between candidates, favors broad
  exploration — good early in a search, or when starting from scratch
  and you don't yet know where the good designs are.
- **Lower** (`0.15`–`0.3`): small, careful steps — good once you're
  already close to a good design and want fine polishing rather than
  large jumps that might overshoot it.
- **Very low** (`<0.1`): barely moves at all; only useful for extremely
  fine last-mile polishing around an already-excellent seed, and even
  then usually not worth going this low — `0.15`–`0.2` is normally
  plenty subtle.

See `de_F_end` below, and the "Phase 1 / Phase 2 workflow" section
further down, for combining an aggressive start with a subtle finish
automatically within a single run.

### DE F end — `de_F_end`
Optional. If set, `de_F` ramps **linearly** from its starting value at
generation 1 to `de_F_end` by the final generation, instead of staying
constant for the whole run. Leave blank/unset (default) for the old
behavior — a constant `de_F` throughout, exactly as if this field didn't
exist. When a ramp is active, the periodic progress line grows an
`F=.. CR=..` tag showing the current values, so you can watch it happen
live rather than take it on faith.

This is the built-in way to do "aggressive broad exploration early,
subtle refinement late" **within one run**, instead of manually running
two separate seeded passes (see the dedicated section below for when
you'd still want to do it manually instead).

### DE CR
Crossover rate: the fraction of dimensions (element lengths/spacings)
that get a new trial value each generation, per candidate; the rest stay
unchanged that round. Default `0.9` (90% of dimensions updated per
trial). This controls *how many* dimensions move each trial, not *how
far* (that's `de_F`) — it matters less for the aggressive-vs-subtle
distinction, but lowering it late in a run (alongside a lower `de_F`)
means each trial only tweaks a handful of dimensions at a time rather
than reshuffling the whole design, which suits fine polishing. Rarely
needs changing on its own; usually left alone or ramped down together
with `de_F`.

### DE CR end — `de_CR_end`
Same idea as `de_F_end`, for `de_CR`: optional linear ramp from the
starting value to `de_CR_end` across the run. Blank/unset (default) means
constant `de_CR`, unchanged from before this field existed.

### Random seed — `seed`
Seeds the random number generator. The same seed + same config always
produces the exact same result (verified: bit-identical across repeated
runs, and across different thread counts). Useful for reproducibility,
and for comparing "what if I only changed this one weight" — keep the
seed fixed while you vary other settings so you're comparing like with
like. Try a few different seed values and compare results if you suspect
the search converged to a mediocre local optimum — DE isn't guaranteed to
find the global best, and different seeds sometimes land on genuinely
different (better or worse) designs.

---

## Phase 1 / Phase 2 workflow: aggressive exploration, then subtle refinement

DE already narrows its own step size naturally over the course of a run
— early on the population is scattered, so mutation steps (which depend
on the *difference* between two other population members) are large; as
individuals converge toward similar designs, those differences shrink on
their own. But if you want that shift to happen faster and more
deliberately than the natural self-narrowing gives you, there are two
ways to do it.

### Option A: automatic, within one run (simplest)

Set `de_F_end`/`de_CR_end` lower than your starting `de_F`/`de_CR`. One
run, one config, the ramp happens on its own:

```
de_F = 0.9        # aggressive start
de_F_end = 0.15   # subtle finish
de_CR = 0.9
de_CR_end = 0.5
generations = 500 # give the ramp room to actually play out
```

Watch the `F=.. CR=..` values in the progress line drop over the course
of the run to confirm it's working as expected.

### Option B: manual, two separate runs

Still useful when you want to actually *look at* what Phase 1 found
before committing to a fine-tuning pass — e.g. widening a bound that
turned out to be limiting, or deciding a floor should be tighter/looser
based on what the broad search actually reached. Also gives you a saved
intermediate result (Phase 1's `.nec`) independent of Phase 2's outcome.

**Phase 1 — broad exploration (find the right neighbourhood):**

| Setting | Value | Why |
|---|---|---|
| `seed_file` | none (or blank) | Start from scratch — full freedom to explore |
| `de_F` | 0.8–1.0 | Large mutation steps |
| `de_CR` | 0.9 | High crossover — most dimensions get a fresh value each trial |
| `pop_size` | 60+ | Wide diversity across the population |
| `generations` | 100–300 | Enough to find a promising region, not necessarily full convergence |
| Floors (`min_gain_dbi` etc.) | loose or off | Let it roam; lock things down in Phase 2 |

**Phase 2 — subtle refinement (polish what you found):**

| Setting | Value | Why |
|---|---|---|
| `seed_file` | Phase 1's `yagi_best.nec` | Refine around the good design, not from scratch |
| `de_F` | 0.2–0.3 | Small, local steps only |
| `seed_jitter_frac` | 0.03–0.08 | Keep most of the population tightly clustered around the seed |
| `seed_random_frac` | 0.05–0.1 | Minimal fresh diversity — you're polishing, not exploring |
| `de_CR` | 0.7–0.9 | Can stay similar; mainly controls how many dimensions move per trial, not how far |
| `generations` | 200–500 | More rounds of fine adjustment now that the space is small |
| Floors | tighten these now | Lock in what Phase 1 found while polishing everything else around it |

You can chain this further into a Phase 3 with `de_F` down around `0.1`
and jitter down around `0.02`, for a genuine staircase of progressively
finer refinement — each stage's seed file already contains everything
the previous stage learned, so nothing is lost between rounds.

**Which option to actually use:** Option A (the automatic ramp) for most
cases — it's simpler and gets you most of the benefit in one run.
Reach for Option B specifically when you want to inspect or adjust
settings *between* the aggressive and subtle stages, not just let them
blend automatically.

---

## 6. Scoring Weights

Every candidate design is scored as one weighted sum, then the search
maximizes that score. There's rarely a design that maxes out every
objective simultaneously, so these weights control the trade-offs.

**Important distinction: rewards vs. floors.** `w_gain`, `w_fb`, and
`w_sidelobe` are *rewards* — they compete with each other in the same
sum, and a high weight on one can still be outweighed if another term
gains more in absolute terms. The floor settings (`min_gain_dbi`,
`min_fb_db`, `min_sidelobe_db`) are *constraints* instead — dropping
below them costs extra on top of the normal reward, regardless of what
else improves. If you need to guarantee something doesn't get worse, use
a floor, not just a bigger weight.

### Priority preset (GUI only)
A quick-select dropdown that fills in the five reward/penalty weight
fields below with a tested starting combination for a stated goal
(Balanced, Max Forward Gain, Max F/B Ratio, Clean Pattern, Max Gain +
Clean Pattern, Tight SWR Match). Picking one doesn't lock you in — the
fields stay fully editable afterward. This doesn't touch the floor
settings, which are design-specific and have no sensible generic preset.

### Gain weight — `w_gain`
Reward for forward gain. Default `1.0` — the baseline everything else is
relative to.

### F/B weight — `w_fb`
Reward for front-to-back ratio, capped internally at 30 dB so a single
sharp, physically fragile null doesn't dominate the score. Default
`0.35`. Raise toward `1.0`–`1.5` if you're getting good gain/SWR but weak
F/B.

### Sidelobe weight — `w_sidelobe`
Reward for sidelobe suppression (also capped at 30 dB). Default `0.2`.
Combine freely with `w_gain`/`w_fb` — e.g. raise `w_gain` and
`w_sidelobe` together for "max gain with clean sidelobes." Remember this
is a reward, not a guarantee (see the floor fields below if you need a
guarantee).

### SWR penalty — `w_swr_penalty`
How hard high VSWR is penalized — a continuous penalty always active
above VSWR 1.0, with a steeper additional penalty once `max_vswr` is
crossed. Default `6.0`.

### Boom penalty — `w_boom_penalty`
How hard exceeding `max_boom_length_m` (or, if set, going under
`min_boom_length_m`) is penalized. Default `40.0` — deliberately much
higher than the reward weights, so it acts like a real constraint rather
than a mild preference. This is the model other floor/constraint-style
penalties below follow.

### Min gain floor (dBi) — `min_gain_dbi`
Optional gain floor. `0` (default) disables it. Unlike `w_gain`, this is
a constraint: dropping below this value costs extra, on top of (not
instead of) the normal per-dB gain reward. Use this when refining an
already-good design for better F/B/sidelobes and you don't want to trade
gain away to get there. Recommended workflow:
1. Run (or seed from) your high-gain design with sidelobe/F-B weights
   low, note the gain at each printed frequency point.
2. Set this to roughly the **lowest** gain value across that table (a
   value at or above your peak is a much stronger ask and fights harder
   against the other objectives).
3. Set **Floor penalty** firmly (see below).
4. **Important:** this is a single floor across the whole band, not a
   per-frequency guarantee — check the full results table afterward, not
   just whether it "passed." A frequency that started well above the
   floor can still drift down toward it even while the floor is
   technically satisfied everywhere.

### Floor penalty — `w_gain_floor_penalty`
How hard dropping below `min_gain_dbi` is punished. Default `3.0`. This
is a *soft* constraint — its effectiveness scales relative to your other
weights. If gain is still slipping below the floor, **raise this**
rather than assuming the floor doesn't work; 20–40 is a reasonable
starting point for something that should feel like a real constraint,
matching the convention `w_boom_penalty` already uses.

### Min F/B floor (dB) — `min_fb_db`
Same idea as the gain floor, for front-to-back ratio. `w_fb` is a reward
and *can* be outweighed by gain/sidelobe improving enough elsewhere in
the score, however high you set it. This is a real constraint instead:
F/B below the floor costs extra regardless of what else improves. `0`
(default) disables it. Pick this from your baseline design's actual F/B
figure, same workflow as the other floors — check the full table
afterward, not just pass/fail.

### FB floor penalty — `w_fb_floor_penalty`
How hard dropping below `min_fb_db` is punished. Default `3.0`, same
soft-constraint caveat as the other floor penalties — raise it (try 20+)
if it's not holding firmly enough. Verified directly: with `w_gain = 5.0`
heavily dominant and `w_fb` reduced to a near-negligible `0.1`, a
`min_fb_db = 25.0` floor with `w_fb_floor_penalty = 25.0` still held F/B
at 26.9–32.8 dB across the band — the floor overrode the reward weighting
entirely, as intended.

### Min sidelobe (dB) — `min_sidelobe_db`
Same idea as the gain floor, for sidelobe suppression. `w_sidelobe` is a
reward and *can* be outweighed by gain/F-B improving enough elsewhere in
the score, however high you set it — DE only ever improves the total
score, so if raising gain nets more than raising the sidelobe weight
costs, the search will do exactly that. This is a real constraint
instead. `0` (default) disables it. Same workflow as the gain floor:
pick this from your baseline design's actual sidelobe suppression figure
(from EZNEC or a prior run), and check the full table afterward rather
than just pass/fail.

### SL floor penalty — `w_sidelobe_floor_penalty`
How hard dropping below `min_sidelobe_db` is punished. Default `3.0`,
same soft-constraint caveat as the gain floor penalty — raise it (try
20+) if it's not holding firmly enough.

---

## 7. Seed (refine an existing design)

### Seed .nec file — `seed_file`
Path to an existing NEC-format deck to refine instead of searching from
scratch. Blank (default) disables seeding — the search starts fully
random. `.EZ` itself is EZNEC's proprietary binary format and can't be
read directly — export to NEC format first (EZNEC: **File > Save
Description As**, choose **NEC** as the file type).

`n_reflectors`/`n_directors` must match what's
actually in the file — elements are identified by position relative to
the driven element (found via the file's source/EX card), not by wire
order, so this works on files EZNEC wrote, not just ones this tool wrote.
If the counts don't match, the log tells you what it found and falls
back to a random search instead of guessing.

The parser accepts comma-separated fields (EZNEC's own export format,
e.g. `GW 1,21,-.5,-.88,...`) as well as space-separated, and is
case-insensitive on card names (`GW`/`gw`/`Gw` all work).

### Jitter frac — `seed_jitter_frac`
How far (as a fraction of each parameter's search range) to randomly
perturb the seed for most of the starting population, so the search
explores around it rather than sitting still. Default `0.15`.

### Random frac — `seed_random_frac`
Fraction of the population left fully random even when seeding, for
diversity — so the search isn't entirely boxed in around the seed.
Default `0.2` (20%).

---

## 8. Output & Optimiser

### Output prefix — `out_prefix`
Base filename for the `.txt` report and `.nec` deck it writes. These are
checkpointed periodically during a run (not only at the very end), so
it's safe to stop a long run early (Cancel, or Ctrl+C at a terminal) and
still have a usable, up-to-date result on disk.

### Optimiser .exe (GUI only)
Path to `yagi_optimize.exe`. Only needs changing if the GUI and console
tool aren't in the same folder.

---

## 9. GUI-only controls

- **Load Config...** — opens an existing `.cfg` file and fills every
  field from it (resets to defaults first, then applies whatever the
  file specifies, same as the console tool's own behavior).
- **Save Config...** — saves the current fields as a `.cfg` without
  running. Useful for keeping a record or hand-editing later.
- **Run Optimiser** — writes the fields to `yagi_gui_run.cfg` and runs
  `yagi_optimize.exe` on it, streaming its output live into the log box
  below.
- **Cancel** — stops a running search. Gives immediate feedback that the
  click registered and includes a safety-net timeout so the UI can't get
  stuck waiting indefinitely.
- **Open Output Folder** — opens the current working folder in Explorer
  so you can grab the `.txt`/`.nec` output.

---

## Quick-reference: recommended values by scenario

**Starting a brand-new design from scratch:** defaults are a reasonable
starting point; widen `Search Ranges` if the search seems boxed in
(scores plateau immediately at generation 1).

**Refining an existing design without losing gain:** set `seed_file`,
`min_gain_dbi` at your baseline's lowest gain, `w_gain_floor_penalty`
20–40.

**Getting it to actually use the full boom length:** set
`min_boom_length_m` close to `max_boom_length_m`, e.g. a 200mm band.

**A K6STI-style tight first-director gap:** `Gap min (all) = 0.06`
(normal), `Dir1-ONLY min/max` set to the tight range instead — never
loosen the general gap bound for this.

**Multiple stacked reflectors with genuinely different pair spacings:**
set `n_reflectors` to 4 or more (stacking is automatic once you have more
than one reflector), and widen `Stack min/max (frac)` enough to cover
both your tightest and widest pair — a narrow range forces all pairs
toward the same spacing.

**Minimum sidelobes without sacrificing gain:** both floors together —
`min_gain_dbi` and `min_sidelobe_db`, each with a firm penalty weight
(20+), plus `w_sidelobe` raised as a reward on top.

**Guaranteed F/B floor while chasing gain:** `min_fb_db` set to your
baseline's F/B figure, `w_fb_floor_penalty` firm (20+), leave `w_gain`
however high you want — the floor holds regardless.

**Aggressive exploration easing into subtle refinement, in one run:**
`de_F = 0.9`, `de_F_end = 0.15`, `de_CR = 0.9`, `de_CR_end = 0.5`, with
enough `generations` (300+) for the ramp to actually play out. See the
dedicated "Phase 1 / Phase 2 workflow" section above for the full
picture, including the manual two-run alternative.
