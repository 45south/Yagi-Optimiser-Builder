# Yagi Optimiser — User Guide

## What it does

Searches for the best Yagi element lengths, spacings, and (for 4+
reflectors) stacking heights — gain, front-to-back ratio, sidelobe
suppression, and VSWR against your target feed impedance — using the
NEC2++ antenna-modelling engine, then reports the winning design and
writes it out as a report and a `.nec` file you can open in EZNEC or
4nec2.

This guide covers day-to-day usage and common workflows. For every
individual setting in detail, see **`FIELD_REFERENCE.md`** — this guide
won't repeat that; it links to the relevant part instead.

## Running it

**Using the GUI (`yagi_gui.exe`):** double-click it, or run it from
Explorer like any normal Windows program. Fill in the fields, or pick a
**Priority preset** to start from a sensible combination for a stated
goal (Max Forward Gain, Max F/B Ratio, Clean Pattern, etc.) — the fields
stay fully editable afterward, picking a preset doesn't lock you in.
Fields showing in **blue text** are ones you've changed away from their
default value, so you can see at a glance what's been customised.
**Reset to Defaults** clears everything back to those starting values in
one click.

Click **Run Optimiser**. Output streams live into the log box at the
bottom, exactly like watching it run in a command window — including a
progress line every 10 generations showing the current score plus the
real worst-case gain/F-B/VSWR/sidelobe/boom-length figures, so you can
watch it actually improve rather than trust an abstract number (see
"What the score means" below). Click **Cancel** to stop a run in
progress — safe to do any time, see "Stopping early" below. When it
finishes, click **Open Output Folder** to find the `.txt` report and
`.nec` file.

Other buttons:

- **Load Config...** — opens an existing `.cfg` file and fills every
  field from it (resets to defaults first, then applies whatever the
  file specifies).
- **Save Config...** — saves the current fields as a `.cfg` without
  running. Useful for keeping a record, or for hand-editing later.
- **Browse...** next to "Seed .nec file" — pick an existing NEC file to
  refine (see "Using an existing design" below).
- **Browse...** next to "Optimiser .exe" — only needed if `yagi_gui.exe`
  and `yagi_optimize.exe` aren't in the same folder.

The GUI doesn't do any of the antenna math itself — it writes a `.cfg`
file from the form and runs `yagi_optimize.exe` for you, so a run takes
exactly as long as it would from the command line.

**Using the command line (`yagi_optimize.exe`):**

1. Open a Command Prompt (or PowerShell) window.
2. `cd` into the folder with `yagi_optimize.exe`.
3. Run it, pointing at a config file:
   ```
   yagi_optimize.exe yagi_example.cfg
   ```

Don't double-click `yagi_optimize.exe` in File Explorer — it needs the
config file as an argument, which only works from a command line. (The
GUI doesn't have this problem — it always supplies one.)

It prints a banner, then an estimated run time, then a progress line
every 10 generations. **Let it run to completion** — the report and
`.nec` file are checkpointed periodically during the run, not only at
the very end, so it's safe to stop early (see below) if you don't want
to wait for the full generation count.

## What comes out

- Printed to the screen: the winning design (each element's length,
  position, and height) and its gain/F-B/VSWR/sidelobe suppression at
  each frequency you checked.
- `<out_prefix>.txt` — the same report, saved to a file.
- `<out_prefix>.nec` — the winning design as a plain-text NEC card deck.
  Opens directly in the free tool 4nec2, and in EZNEC Pro/2+ via File >
  Open with "NEC" selected as the file type. `out_prefix` is set in the
  config file.

## What the score means

The printed "score" isn't a physical quantity — it's a made-up composite
of everything you've weighted (gain + F/B + sidelobe rewards, minus VSWR
and any floor/ceiling penalties), and there's no way to know in advance
what the best possible score for a given setup actually is. Antenna
performance comes out of solving Maxwell's equations numerically for a
given shape; there's no formula that tells you the answer ahead of time,
the same way there's no formula for "the best possible chess move."

Treat it as a **relative trend within one run**, not an absolute measure:
climbing means still improving, flat for a long stretch means converged
(more generations won't help). Don't compare score values *between*
different configs or weight combinations — they're not on the same
scale unless the weights are identical. The actual results that matter
are the printed Gain/F-B/VSWR/Sidelobe figures, not the score itself.

## Stopping early

`<out_prefix>.txt`/`.nec` are checkpointed periodically (roughly every
50 generations on longer runs), not only at the very end — so Cancel (in
the GUI) or Ctrl+C (at a terminal) always leaves you with a usable,
up-to-date result rather than losing the whole run. If the progress line
has been printing the same score for a long stretch, there's usually
nothing to gain by waiting — checking the trend and stopping early is a
normal, safe way to work, not a compromise.

## Common workflows

### Refining without losing gain (or F/B, or sidelobes)

If you've got a design that's excellent on one objective and want to
improve another *without* sacrificing the first, don't just raise the
other weight and hope — set a **floor** (or, for VSWR, a **ceiling**).
Weights (`w_gain`, `w_fb`, `w_sidelobe`) are rewards that compete with
each other in the same sum, and a high weight on one can still be
outweighed if another improves more in absolute terms. Floors
(`min_gain_dbi`, `min_fb_db`, `min_sidelobe_db`) and the VSWR ceiling
(`max_vswr` + `w_vswr_ceiling_penalty`) are real constraints instead —
dropping below (or above, for VSWR) them costs extra regardless of what
else improves.

Recommended workflow:

1. Run (or seed from) your good design with the *other* weights left
   low, note the relevant figure at each frequency point in the printed
   table.
2. Set the floor to roughly the **worst** value across that table (not
   the best) — protects the band's weak point specifically.
3. Set the matching penalty weight firmly — 20–40 is a reasonable
   starting point for something that should feel like a real constraint.
4. Seed from your original design, raise the weights you actually want
   to improve, and run.
5. **Check the full per-frequency table afterward, not just whether it
   technically passed.** A floor is one value across the whole band, not
   a per-frequency guarantee — it stops the worst point from crashing,
   but a frequency that started well above the floor can still drift
   down toward it.

Full field details: `FIELD_REFERENCE.md`'s "Scoring Weights" section.

### Using an existing design as a starting point

If you already have a design (from EZNEC, a previous run of this tool,
or anywhere else) and want the optimiser to refine it rather than search
from scratch:

1. Get it into NEC format. In EZNEC: **File > Save Description As**,
   choose **NEC** as the file type. (EZNEC's own `.EZ` format is
   proprietary binary and can't be read directly — the NEC export is the
   bridge.)
2. In your config, set `seed_file = path/to/that.nec`.
3. Set `n_reflectors`/`n_directors` to match what's actually in that
   file — the tool identifies each wire by its position relative to the
   driven element (found from the file's source/EX card), not by wire
   order, so it works on files EZNEC wrote, not just ones this tool
   wrote. If the counts don't match what it finds, it'll tell you and
   fall back to a random search instead of silently doing the wrong
   thing.
4. Run it as normal. Most of the starting population will be small
   variations on your design (`seed_jitter_frac`); a portion stays fully
   random for diversity (`seed_random_frac`).

### Getting it to use the full boom

`max_boom_length_m` only ever *caps* the design — it never encourages
the search to actually use that space. If gain and F/B are weighted
heavily, the optimiser will happily shrink the boom if a shorter,
tighter geometry scores well, since nothing rewards spreading elements
out — and this often costs you on sidelobes, since longer booms with
more spacing generally give better pattern control.

Two changes fix this together: set `min_boom_length_m` close to your
target (e.g. max `4.2`, min `4.0`), and raise `w_sidelobe` relative to
`w_gain`/`w_fb` — that's the actual lever for cleaner sidelobes, which
keeps losing out if gain/F-B dominate regardless of boom length.

### Aggressive exploration, then subtle refinement

Two ways to get "broad search early, fine polishing late": set
`de_F_end`/`de_CR_end` lower than your starting `de_F`/`de_CR` for an
automatic ramp within one run, or manually run two seeded passes (broad
settings first, then a tightly-seeded refinement pass). Full detail,
including recommended values for both approaches, in
`FIELD_REFERENCE.md`'s dedicated "Phase 1 / Phase 2 workflow" section.

### Modelling real wire loss

By default every wire is a perfect conductor — set `wire_resistivity_ohm_m`
(or pick a material from the GUI's Wire Loss dropdown: copper, two
aluminium alloys, tin, zinc, or your own value) to model real conductor
loss instead. `0` (the default) changes nothing from how every earlier
version of this tool behaved. Full detail and preset values in
`FIELD_REFERENCE.md`'s "Wire Loss" section.

### Combining multiple goals at once

All the objectives are scored together every run. There's rarely a
design that maxes out everything simultaneously, so raising one weight
typically trades off against another — for "maximum gain with minimum
sidelobes," raise both `w_gain` and `w_sidelobe` and see what the search
settles on, then check the printed table to see the actual trade-off it
found and adjust from there.

## How long it takes

Roughly `pop_size × (generations + 1) × n_freq_pts` NEC calculations, at
a few milliseconds each. The example config (60 × 251 × 5 ≈ 75,000)
takes about 5 minutes on a typical desktop.

For quick experiments while tuning weights or bounds, drop `pop_size`
and `generations` (e.g. 20–30 / 40–60) to get a rough answer in seconds,
then scale back up for a final run.

Running more than one instance at once on the same machine? Set
`max_threads` on each (roughly your core count divided by the number of
instances) to avoid oversubscription — see `FIELD_REFERENCE.md`.

## Typical workflow

1. Start from `yagi_example.cfg`: set your frequency range, element
   diameters, and `n_reflectors`/`n_directors` (4 or more reflectors
   stacks automatically — nothing else to set for that).
2. Do a quick low-`pop_size`/`generations` run to sanity-check it
   produces a sensible-looking design.
3. If F/B is weak, raise `w_fb`. If VSWR is stuck too high, raise
   `w_swr_penalty` (or a firm `w_vswr_ceiling_penalty` if it needs to
   hold hard), or check your `refl_len_*`/`gap_*` bounds are wide enough
   to include a better match.
4. Once you're happy with the trade-off, run it at full `pop_size`/
   `generations` for the final design — consider a `de_F_end` ramp for a
   more thorough final pass.
5. Open the `.nec` output in EZNEC or 4nec2 to inspect the pattern and
   double-check before you build it. If you're checking it over **real**
   ground rather than free space, be aware of the genuine gain null at
   exactly 0° elevation for a horizontal antenna — plot a small nonzero
   elevation angle, or use an elevation plot, if EZNEC reports "no sky
   wave" there.
