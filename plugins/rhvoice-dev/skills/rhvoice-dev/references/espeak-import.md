# Importing eSpeak NG rules into a foma g2p

This method was worked out over many imports of upstream eSpeak NG
`<lang>_rules`/`<lang>_list` changes into an RHVoice module whose exceptions cite
eSpeak line numbers. It fits any language that mirrors eSpeak rules in foma. The
rule snippets below are schematic.

## Contents
1. Setting up eSpeak NG as a reference
2. Which new rules actually matter
3. Citations and renumbering
4. Precedence traps (eSpeak semantics)
5. Comparing readings: relation pitfalls
6. QA bar for an import
7. Deduplication

## 1. Setting up eSpeak NG as a reference

- Build without extras (about 1 minute):

  ```
  cmake -B build -DUSE_MBROLA=OFF -DUSE_LIBSONIC=OFF -DUSE_LIBPCAUDIO=OFF -DUSE_ASYNC=OFF
  cmake --build build -j
  ```

- Recompile the dictionary after every pull. In `dictsource`:
  `ESPEAK_DATA_PATH=../build ../build/src/espeak-ng --compile=<lang>`. A stale
  `<lang>_dict` gives wrong references.
- **Old-dictionary copy** (to compare eSpeak before and after an upstream commit):
  the `ESPEAK_DATA_PATH` directory must *contain* `espeak-ng-data/`. Copying
  `espeak-ng-data` *as* the directory silently writes the dict to the wrong place.
  Read with `--path=<copy>`.
- `espeak-ng -X` prints one `Translate` line per pass. For words with a suffix
  eSpeak strips, it cuts the ending off and re-reads the stem, so the final
  reading is not a whole-word rule match.
- **Upstream branches get rebased.** Pinned hashes may disappear from a fresh clone.
  Keep an old→new hash table in the audit notes, and check content identity (the
  rule lines themselves) before trusting a new pin.
- eSpeak on 650k words takes ~30 min, RHVoice ~2 min. Start eSpeak first.

## 2. Which new rules actually matter

Compile the previous commit's rules into a copy of the data and compare old eSpeak,
new eSpeak and RHVoice on stem + context × many tails. Often several of the new
rules need no foma change, because RHVoice already reads them the new way. A
change to the `_list` file only (word flags such as `$unstressend`) can also need
no network change at all. Record that in the audit instead of touching the
grammar.

## 3. Citations and renumbering

If exceptions cite `<lang>_rules:N`:

- Every inserted upstream line shifts all later citations: single citations,
  lists (`N, M, …`), prose ("N and M") and line columns of TSV fixtures.
  **Renumber every number in a list, not only the first.** An earlier import that
  missed this left 11 lists stale.
- Use a difflib line mapping between the old and new rules file, assert it with a
  formula, and run a citation checker that compares each cited line's text.
- When counting insertion points from a git diff, the `@@ … @@` function-context
  text is **not** part of the hunk. Count from the hunk body.
- Audit notes of earlier imports keep their old numbers.

## 4. Precedence traps (eSpeak semantics)

- **A longer left context outranks a new shorter rule.** For example, an existing
  `XYab) c (d` beats a new `ab) cc (de`, so the RHVoice override for the new rule
  must exclude `XY`, with enumerated contexts and not a complement. Compare the
  eSpeak reading of every real first member of a compound.
- **In `_list` files the LATER duplicate wins** (compiledict prepends to the hash
  chain), so a new line can have no effect because an older one shadows it, or the
  reverse. Verify by deleting a line in a copy.
- A higher-precedence upstream rule must also remove any seam marker your own
  exception layer inserted, not only the marker of its own layer.
- Some multigraph rules consume part of the next match: a rule that reads a seam
  plus the first letter of a digraph. Every such stem needs a matching "consumed"
  guard, or a later general rule inserts a second seam. Check the guard **before**
  starting a 15-minute build.
- RHVoice applies a rule in suffix-stripped forms too. Do not copy eSpeak's
  behaviour there.

## 5. Comparing readings: relation pitfalls

A checker that classifies differences as "assimilation" can hide real
regressions:

- a kept seam vs a fused affricate,
- a single digraph phone vs two separate phones,
- a final vowel collapsing before a same-vowel word.

Use a strict phone-by-phone relation and allow only differences you can name:
voicing only before an obstruent, nasal place only before a consonant, length
last.

## 6. QA bar for an import

- One real compound per rule × every inflected form the language has (cases,
  number, possessive, …) × common prefixes × Title/UPPER case. Expect
  reading(left) ++ reading(free right form), checked against eSpeak and
  old-vs-new.
- An old-vs-new sweep over fixtures, the frequency word list and all voice prompt
  texts available (hundreds of thousands to millions of words).
- Run the tokenizer and other checks the module has, such as functional and
  totality proofs. See [foma.md](foma.md#5-proofs-totality-functionality-equivalence).
- For a list change that affects dates or numbers, run a sentence matrix (month
  spellings × days × words × sentence shapes). Run eSpeak one process per
  sentence so its sentence splitting stays aligned.

## 7. Deduplication

When an upstream rule repeats an older line, or lies inside an existing context,
add a citation at that context instead of a second entry. Merge context lines
that name one alternative twice. Alternatives merely covered by a shorter one are
distinct upstream rules; keep them unless the maintainer says otherwise.
