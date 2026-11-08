# Training RHVoice HTS voices (voice-building-utils / vbu)

Notes on preparing corpora and running the HTS training scripts that ship with
RHVoice (`src/scripts/general/vbu`, `voice-building-utils`, `src/scripts/hts`).
Check your own tree's version: some of the bugs below are fixed in newer copies.

## Contents
1. Corpus preparation and sync checks
2. Large corpora: recording numbering and HCompV precision
3. f0 range and vocal fry
4. problematic_files and segmentation quality
5. VAD-initialised silence (`seg_init_sil`)
6. Multicore training
7. Praat 7

## 1. Corpus preparation and sync checks

A common corpus layout is `metadata.csv` (`id | text | normalized text`, pipe
separated, no header) plus `wav/<id>.wav`. Write `prompts-<voice>.ssml` from the
id and normalized-text columns:

```xml
<speak xml:lang="xx">
<s xml:id="0b4c…">…</s>
</speak>
```

Use one `<s>` per row, in metadata order, and escape `& < > "`.

Cheap checks before training (seconds, no workspace needed):

- ids ↔ wav files are 1:1 and unique;
- the SSML parsed the way vbu does (`doc.findall("s")`) gives back the metadata
  order and text;
- all WAV headers are valid (96 kHz/24-bit `WAVE_FORMAT_EXTENSIBLE` is common);
- `RHVoice-transcribe-sentences -b ssil prompts.ssml out.txt` gives exactly one line
  per prompt and no errors;
- a duration-vs-letter-count fit (e.g. 0.8 s + 0.075 s per letter) has no unexplained
  outliers. Long outliers are usually name lists or trailing digital silence, and
  fast ones spelled-out years.

Also look for typos in the text column, digits or abbreviations the speaker
expanded differently (initials read as full first names, acronyms read as one
syllable), and parenthesised material the speaker skipped.

Speaker gender and age matter for `training.cfg`. A quick autocorrelation pitch
sample of ~40 files gives a hint (median ~115 Hz male, ~175–195 Hz female), but
ask the user or check the corpus documentation rather than assuming.

## 2. Large corpora

- **More than 9,999 prompts**: older vbu/voice-building-utils named imported
  recordings `RHVoice_<Speaker>_{:04}` and later paired sentence N with the N-th
  file in *lexical* order. Above 9,999 prompts most sentences were silently
  aligned to the wrong audio. Fixed versions pad to `max(4, digits of the prompt
  count)`. A workspace imported before the fix must be re-imported from a clean
  workspace.
- **HCompV single-precision drift**: HTK 3.4.1 HCompV sums frames in single
  precision. At ~12M frames (17 h at 5 ms) the c0 variance came out negative and
  was clamped to 0, and every file in `HERest_0` logged `WARNING [-7324] StepBack …
  bad data or over pruning`. Measured c0 variance drift: 0.2% at 1.0M frames, 2.6%
  at 3.1M, 6% at 4.0M, 33% at 6.1M. Fix: give HCompV an evenly spaced subset of at
  most ~1M frames (`htk/hcompv.scp`) and leave HERest the full set.

## 3. f0 range and vocal fry

`guess-f0-range` sets `lower_f0 = exp(mean − 3·sd)` of the RMVPE log-f0, and
`extract_with_rmvpe` zeroes every frame below `lower_f0`, so creaky voice (vocal
fry) turns unvoiced. If the user wants fry modelled, set `lower_f0` by hand after
guess-f0-range (≈40 Hz). RMVPE cannot go below 31.7 Hz, and at 24 kHz the
CheapTrick FFT stays at 2048 down to ~36 Hz. Look at a few fry files first. Store
the guessed values in `training.cfg` so later runs can skip the guess.

## 4. problematic_files and segmentation quality

`problematic_files_N.txt` lists files where HVite alignment round N needed beam
widening ("No tokens survived … at beam 250"). Column 2 is the number of retries.
The frame rate is 5 ms, so a 5-state phone lasts at least 25 ms. Fast speech with
many minimum-length phones shows up here without any error in the text.

To find real misreadings:

1. Run ASR on just those files (e.g. Whisper large-v3 via faster-whisper) and
   compare with the prompt text.
2. Confirm with the voice's own models: see
   [htk-alignment-diagnosis.md](htk-alignment-diagnosis.md).
3. Fix the prompt (text or `<phoneme>`, see [ssml-phoneme.md](ssml-phoneme.md)),
   realign old vs new labels with the original `-t 250 150 1000`, and count
   "No tokens survived".

Recurring patterns in the data:

- cross-word degemination (two identical consonants across a word boundary said
  as one),
- sibilant assimilation across a word boundary,
- dropped word-final consonants in fast speech,
- a vowel squeezed to the 5-frame minimum before a glide (an HMM floor effect, not
  a missing rule).

Decide with a native speaker whether such a pattern is a language-module rule or
a one-off.

## 5. VAD-initialised silence (`seg_init_sil`)

`"seg_init_sil": true` in training.cfg runs webrtcvad on the 16 kHz audio (30 ms
window every 5 ms, `seg_vad_mode` default 0), trains a 1-state `sil` HMM on the VAD
silences and copies it into every state of the initial `pau`. Only the flat start
changes.

Measured on one 6.5 h corpus:

- The same files ended up problematic, and the final likelihood was equal.
- Silence labels were cleaner. Speech onset moved later in many stop-initial files,
  so the median initial stop went from 95 to 55 ms. Spurious 25 ms internal
  pauses disappeared, and 45 ms real pauses appeared.
- VAD mode 0 calls only ~20% of silence frames silence but almost never errs on
  speech. Mode 3 catches ~90% of silence but labels 43% of fricative and 59% of
  stop frames as silence. **Do not use mode 3.**

## 6. Multicore training

If you use a multicore patch for the HTS scripts (parallel HERest, align, Praat),
these choices kept serial and parallel runs comparable:

- every `align` round covers all recordings, so earlier failures still show up in
  `problematic_files_3/4`, which corpus error hunting relies on;
- only HERest gets the filtered scp;
- cache the Praat command per process, since pools pickle the task object for
  every file;
- no artificial sleep between Praat runs (8 parallel Praat 7 runs were
  byte-identical without it);
- leave HTK stderr on the terminal;
- without a `cores` key, leave a hand-set `$NUMPROC` alone.

Expected test result on a small synthetic corpus: `--cores 1` is byte-identical to
the serial scripts, and features and cmp files are byte-identical at any core
count. Segmentation and models differ only by tiny float effects, because HTK
accumulators are summed in a different order. A common limit: parallel HERest
assumes ~3 GiB per worker, so a 30 GB machine runs at most ~7 workers.

## 7. Praat 7

Praat 7.0 refuses every "Save as…" and system command in `--run` scripts unless
`--FULL-TRUST` is given (`…is not allowed without --FULL-TRUST`). This breaks
`import-audio` (import.praat saves raw+wav). The pitch printing scripts only read
and still work. Praat 6.4.x rejects the option ("Unrecognized command line
option", exit 255) and 6.6.30 accepts it.

Fix in the scripts: add `--FULL-TRUST` only when `praat --help` lists it.
trim/split/resample scripts run by hand need the option too. (Behaviour taken
from the v7.0.02 source, `melder_trust.cpp`. The detection fix was tested on
6.4.67.)
