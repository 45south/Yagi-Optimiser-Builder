# Yagi Optimiser — Changelog

A build-by-build record of what changed and why. Version numbers follow
`v1.0.NNNN`, where `NNNN` is the build number shown in both the console
banner and the GUI title bar.

**A note on accuracy:** builds 12 onward below are a precise, real-time
record — each one corresponds to an actual, distinct release. Builds
1–11 are a **best-effort reconstruction**, put together retroactively
after realising the build number wasn't being incremented as it should
have been from the start. The list of what changed in that early period
is accurate; the exact boundary between which build number contained
which specific fix is not — treat build numbers below 12 as approximate.

---

## Build 15
- Fixed a layout bug: the Scoring Weights, Seed, and Output group boxes
  had moved (to make room for the DE F end/CR end row added in build 14)
  but the individual fields inside them hadn't — leaving them rendered
  outside their own group box borders.

## Build 14
- Added `de_F_end`/`de_CR_end` — an optional linear ramp for the DE
  mutation/crossover parameters across a run (aggressive broad
  exploration early, subtle fine-tuning late), automating what
  previously required two manually-seeded runs. Blank/unset by default,
  fully backward compatible.
- Progress line grows an `F=.. CR=..` tag whenever a ramp is active.

## Build 13
- Fixed the blue banner's width, which had accidentally shrunk when
  text padding was added in build 12 (the fix moved the whole control
  rectangle instead of just the text within it).
- Fixed remaining US-spelling ("Optimizer") instances: the Run button,
  the Output group label, the Optimiser .exe field label, and matching
  text written into generated `.cfg` files.
- The exported `.nec` file's description (shown as the antenna name in
  EZNEC) is now generated dynamically from the actual band and element
  count, e.g. `85-93 MHz - 11 el. yagi`, instead of a generic string.

## Build 12
- Version string format changed to `v1.0.NNNN` (zero-padded build
  number as the third version segment) instead of `v1.0 (build N)`.
- Boom length added to the periodic progress line during a run.
- GUI: text in any field that differs from its default value now
  renders in blue, so changed settings are visible at a glance.
- GUI: banner padding added (text no longer flush against the window
  edges) and the URL font size increased slightly.

## Builds 2–11 (reconstructed)
In roughly this order, though exact build boundaries within this range
aren't precisely known:
- **Reset to Defaults** button added to the GUI.
- Fixed a bug in the exported `.nec` file: the `RP` (radiation pattern)
  card was missing two required fields, which caused EZNEC to throw a
  runtime error when trying to view the pattern plot or wire geometry.
  The exported file's plotted elevation is now also the design's actual
  pattern peak (found by a small scan), not always assumed to be the
  horizon — which matters specifically for stacked-reflector designs.
- **Wire loss** modelling added — a `wire_resistivity_ohm_m` setting
  (0 = lossless, unchanged from all earlier behaviour) applying real
  conductor resistivity via NEC's segment-conductivity loading. Found
  and fixed a related bug in the same pass: the gain figure was
  computed as "directive gain," which by definition excludes ohmic
  losses, so wire loss initially had no visible effect at all until
  switched to "power gain."
- Material presets added for wire loss: Copper, two aluminium alloys
  (6061-T6 and 6060-T5, with the former matching EZNEC's own default),
  Tin, Zinc, plus a User Defined option.
- **VSWR ceiling** (`w_vswr_ceiling_penalty`) — a genuinely firm,
  independently-weighted penalty for exceeding `max_vswr`, replacing an
  earlier fixed, weaker multiplier that could still be outweighed when
  chasing gain hard.
- Live progress figures (worst-case gain/F-B/VSWR/sidelobe) added
  alongside the raw score in the periodic progress line, so real
  physical figures are visible during a run, not just the abstract
  score.
- Fixed a GUI bug where the log box would silently stop updating on long
  runs (a Win32 `EDIT` control's default text-length cap being hit
  without ever being explicitly raised) while the underlying run and its
  checkpoint files continued completely normally.
- Hardened the pre-search NEC library "warm-up" step (single-threaded
  priming of internal one-time setup code, to avoid a data race under
  multi-core parallel search) — expanded from one fixed geometry to
  several diverse ones, in response to an intermittent heap-corruption
  crash (`STATUS_HEAP_CORRUPTION`) that couldn't be conclusively
  reproduced but was consistent with this class of issue.
- **`max_threads`** setting added, to cap CPU thread usage — useful when
  running more than one instance at once, to avoid oversubscription.
- Fixed a real, confirmed bug: reflector pairs in a stacked (4+
  reflector) design could converge to the same spacing value, placing
  two reflectors at the exact same physical position (reported directly
  as visibly overlapping elements). Reworked the pair-spacing search to
  make this structurally impossible rather than just unlikely.

## Build 1
- First version-numbered release. Added the in-app banner (console
  startup banner and GUI title bar) showing app name, version, build
  number, author, and GitHub URL.

## Before build 1 (unnumbered)
Everything from the initial console optimiser and DE search through the
GUI's first working version, multi-reflector stacking, the sidelobe
metric fix, and the elevation-peak fix — see `README.md`'s "How it
works" section for what these cover. No version numbering existed yet,
so there's no meaningful build-by-build breakdown for this period.
