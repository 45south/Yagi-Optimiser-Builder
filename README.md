# Yagi Optimiser (NEC2++ + Differential Evolution)

**v1.0.0015** — Dave Headland — https://github.com/45south

Searches element lengths, spacings, and (optionally) stacking heights for
a Yagi — multiple reflectors, one driven element, multiple directors —
to maximise gain, front-to-back ratio, and sidelobe suppression while
holding VSWR near a target feed impedance, across a frequency range.
Uses NEC2++ (`libnecpp`) as the EM solver — the same method-of-moments
math EZNEC is built on — not EZNEC itself, since EZNEC has no
scripting/automation interface.

Two ways to use it:
- **`yagi_gui.exe`** — a native Windows dialog: fields and dropdowns
  instead of hand-editing config files, a live log while it runs, preset
  buttons for common goals. Windows/MinGW only.
- **`yagi_optimize.exe`** — the same engine from the command line, with
  a plain-text `.cfg` file. Builds on Linux/macOS too, not just Windows.

The GUI doesn't do any of the antenna math itself — it just writes a
`.cfg` file from the form and runs the console tool as a child process,
streaming its output live. Both give identical results for identical
settings.

## Documentation

- **This file** — build instructions, quick start, how it works
- **[USER_GUIDE.md](USER_GUIDE.md)** — day-to-day usage, workflows, what
  the output means
- **[FIELD_REFERENCE.md](FIELD_REFERENCE.md)** — every setting in both
  the GUI and the `.cfg` file, in detail, with recommended values —
  the authoritative reference; if this README and that file ever
  disagree, trust `FIELD_REFERENCE.md`

## Build

Needs `git`, `cmake`, and a C++17-capable `g++` (MinGW-w64 is fine).

```
make necpp     # one-time: clones and builds the NEC2++ library into ./necpp
make           # builds yagi_optimize (both it and yagi_gui.exe together, on Windows)
```

On Linux/macOS, plain `make` only builds the console tool (the GUI needs
`windows.h`, so it's skipped automatically there rather than failing).
On Windows, `make` builds both together in one step; `make gui` on its
own rebuilds just the GUI, e.g. after only editing GUI source files.

`make clean` removes build output; `make distclean` also removes the
cloned `necpp/` tree. After a `clean`, plain `make` rebuilds everything
correctly on Windows (both targets together) — but `make gui` on its own
only rebuilds the GUI, so running just that after a `clean` leaves
`yagi_optimize.exe` deleted and not rebuilt. Use plain `make` unless
you're deliberately rebuilding just one piece.

If you already have necpp built elsewhere: `make NECPP_DIR=/path/to/necpp`.

By hand, same steps the Makefile runs:

```
git clone https://github.com/tmolteno/necpp.git
cd necpp
cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF
cmake --build build --target necpp_static -j4
cd ..

gcc -O2 -Wall -fopenmp -c yagi_optimize.c -I necpp/src -o yagi_optimize.o
g++ -O2 -Wall -fopenmp -static -o yagi_optimize yagi_optimize.o necpp/build/src/libnecpp.a -lm
```

On Windows/MSYS2, `make necpp` picks the MinGW Makefiles cmake generator
automatically; by hand, add `-G "MinGW Makefiles"` to the first `cmake`
line. The GUI (`yagi_gui.c`/`.rc`/`resource.h`) is plain Win32 C with no
necpp dependency — see the `make gui` recipe in the `Makefile` if
building it by hand.

## Quick start

**GUI:** run `yagi_gui.exe`, fill in the fields (or pick a **Priority
preset** to start from a sensible combination), click **Run Optimiser**.
Watch the log. **Reset to Defaults** clears everything back to sensible
starting values; fields showing in blue are ones you've changed from
their default, so you can see at a glance what's been customised.

**Command line:**
```
yagi_optimize.exe yagi_example.cfg
```
Edit `yagi_example.cfg` (plain `key = value` text) for your design —
every key is documented in `FIELD_REFERENCE.md`.

Don't double-click `yagi_optimize.exe` in File Explorer — it needs the
config file as an argument, which only works from a command line (or via
the GUI, which always supplies one for you).

## What comes out

Printed progress every 10 generations — current score, worst-case
gain/F-B/VSWR/sidelobe across your frequency band, and boom length — plus
two files: `<out_prefix>.txt` (the same report) and `<out_prefix>.nec` (a
plain-text NEC2 deck of the winning design).

The `.nec` file opens directly in the free tool **4nec2**, and in EZNEC
Pro/2+ via File > Open with "NEC" selected as the file type. It's also
what `seed_file` reads back in if you want to refine that design
further — including files exported from EZNEC itself (**not** `.EZ`,
which is a proprietary binary format; use File > Save Description As,
choose NEC as the file type).

Both output files are checkpointed periodically during a run, not only
at the end — safe to stop early (Cancel in the GUI, Ctrl+C at a
terminal) and still have a usable, up-to-date result.

## Headline features

A quick map of what's available — see `FIELD_REFERENCE.md` for the full
detail on every one of these:

- **Multiple reflectors, correctly stacked.** More than one reflector is
  always arranged as a real design would (stacked at different heights,
  sharing one boom position) — this is automatic based on how many
  reflectors you ask for, not a separate setting. Each symmetric pair
  gets its own independent spacing, so tightly-coupled inner pairs and
  widely-spaced outer pairs are both representable.
- **Seed from an existing design** (yours or from EZNEC) and refine it,
  instead of always searching from scratch.
- **Hard floors and ceilings**, not just weighted preferences — a
  minimum gain, minimum F/B, minimum sidelobe suppression, or VSWR
  ceiling that the search treats as real constraints, holding firm even
  when other objectives are pushed hard.
- **Wire loss** — model real conductor resistivity (copper, a couple of
  aluminium alloys, or your own value) instead of assuming perfect
  conductors.
- **A first-director-only gap range**, separate from the general
  element spacing — for designs that deliberately use one unusually
  tight gap near the feed while keeping the rest of the boom
  conventional.
- **An automatic aggressive-then-subtle search ramp** (`de_F_end`/
  `de_CR_end`) — broad exploration early in a run, fine polishing late,
  within a single run instead of needing two manually-seeded passes.
- **Multi-threaded** (OpenMP) — uses all available CPU cores by default;
  cap it with `max_threads` if running more than one instance at once.

## How it works

All-free-space model (no ground) — standard for comparing Yagi designs
on gain/F-B/impedance; add height/ground effects separately once you've
picked a design (real-ground modelling has its own quirks, like a
genuine gain null at exactly 0° elevation — see `USER_GUIDE.md` if you
hit that in EZNEC). Elements are modelled as single wires, driven element
fed at its centre. Search is a differential-evolution algorithm: a
population of candidate designs is repeatedly mutated and the better
performer kept, over many generations — this handles a noisy,
non-convex, multi-objective search far better than brute-force gridding.

Score per candidate combines weighted gain, F/B, and sidelobe
suppression, minus penalties for VSWR above target, pattern squint, boom
length outside your limits, and any floor/ceiling violations. There's no
knowable "perfect score" to compare against — it's a relative signal for
whether a run is still improving, not an absolute measure (see
`USER_GUIDE.md` if this comes up).

Gain/F-B/sidelobe are evaluated at the pattern's *actual* peak elevation
for each design, found by a small scan rather than assumed to sit at the
horizon — this matters specifically for stacked-reflector designs, which
aren't vertically symmetric and can have a genuinely tilted peak.

## A note on how this was built

This tool, its GUI, and every fix described in `FIELD_REFERENCE.md`'s
version history came out of an extended, iterative conversation — real
bugs were found (and fixed) specifically because real designs were run
through it and the results didn't match what EZNEC reported for the same
geometry. If something looks wrong, it's worth reporting exactly that
way: what you expected, what you got, and ideally a comparison against
an independent source (EZNEC, 4nec2, hand calculation). That's how every
fix so far has actually been found.
