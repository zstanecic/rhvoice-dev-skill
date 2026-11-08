# foma for RHVoice: traps, proofs and size control

Seen with foma 0.9.18 in several RHVoice language modules. The traps are generic,
and the rule snippets are schematic.

## Contents
1. Silent syntax traps
2. Rules that leave words without output ("G2p failed", "Tokenization failed")
3. Compiling: timing and the "edit during build" trap
4. Testing a network in foma itself
5. Proofs: totality, functionality, equivalence
6. Size control
7. Very large word-symbol alphabets
8. Where a fix belongs (layering)

## 1. Silent syntax traps

- A bare multi-letter token such as `(en | s)` is **one multichar symbol** `en`. It
  enters the alphabet and changes tokenisation, so every word containing "en" can
  lose its output. Write `{en}` for a letter sequence.
- `\X` is the single-symbol complement and `~X` the language complement. For a
  transducer, complement its upper side: `~[T.u]`.
- `define X` must have its expression on the same line. Otherwise foma pops the
  stack and aborts.
- `.#.` works only in rule contexts, not inside a subexpression like `[X] - [a .#.]`.
- If a build script regenerates `.foma` files from generator scripts
  (`generate-*.py`), edit the generator, not the generated file.
- A complement left context such as `.#. ~[?* Stems] _ R` silently compiles to the
  plain rule. Enumerate the contexts instead.

## 2. Rules that leave words without output

RHVoice translates with an unweighted FST. If a word has no path through the
network, the word, and with `G2p failed` the whole sentence, is dropped.

- **`?` in a left context of an insertion rule.** In `[..] -> M || L _ R, …`, a `?`
  (or `\x`) in a LEFT context also matches the marker `M` that another context of
  the same rule has just inserted. The rule then has no output for that input. Fix
  it by excluding the marker: `[? - s - "MARKER"]`. A `?` in a right context did not
  show the fault. Reproduce suspicions in a 3-line grammar before starting a long
  build.
- **Parallel `@->` rules whose context fails** leave the input untagged. A
  tokenizer then reports `Tokenization failed`. Example: splitting a joined +/- sign
  inside a stage with a context on the signed number. Do such splits as their own
  composed step.
- **Tokenizer shapes that `numbers.fst` cannot read**: RHVoice drops a `num` token
  that has no path in numbers.fst and writes nothing. See
  [front-end-testing.md](front-end-testing.md), "Finding dropped tokens".
- **Output that must be one symbol**: when a rule maps to a multichar symbol that a
  later FST reads as a unit (e.g. a month name key), make sure the output really is
  the single symbol. `X .o. {june}` spelled the word letter by letter, while
  `X .o. {june}:june` gave the symbol.

## 3. Compiling: timing and the "edit during build" trap

- Foma reads a sourced file **while it compiles**, not up front. Editing any `.foma`
  file during a build, a `test functional` run or the totality test makes foma
  abort with `Unknown command '[' on line N`, on a line that does not exist. Finish
  all edits first, or work on a `cp -a` copy of the source directory.
- Big networks (a 40–50 MB g2p) take 12–25 minutes. Run builds in the background.
  Two builds can run in parallel on a multicore machine.
- Build scripts are sometimes tracked without the executable bit. Run them with
  `bash foma2fst <name>` instead of `chmod`-ing them in git.

## 4. Testing a network in foma itself

- `apply down` and `flookup` tokenise the **input** by longest match over the whole
  alphabet, including multichar output symbols. A word containing such a letter
  pair (`ts`, `nj`, `ll`, … depending on the phone set) prints `???` although
  RHVoice, which feeds single letters, transcribes it fine. One lexicon check
  produced 2,195 false failures out of 150,778 words this way.
  For whole word lists, use `RHVoice-transcribe-sentences -v`, or export
  `write att` and walk the ATT file with a small applier that feeds one character
  at a time.
- Ablation (finding what makes a network big): write variants of the file without
  the final `regex`, then run `printf 'source X\nregex WordG2P;\n' | foma` for
  several variants in parallel and compare sizes.

## 5. Proofs: totality, functionality, equivalence

- **Totality** (every word has a reading). `test functional` alone does not catch
  inputs with zero readings.

  ```
  source g2p.foma
  define G ;
  define L [<all letters>] ;
  regex [L+ [[%-|%'] L+]*] - G.u ;
  test null
  print shortest-string
  ```

  `test null` must print 1. If it does not, `print shortest-string` shows the
  shortest word without output. This takes about as long as the build.
- **Functionality**: `test functional` should hold for a g2p that must give one
  reading per word.
- **Equivalence of a rewrite**: cut the define out of both versions, rename one,
  and run `test equivalent`. When the layer uses an internal marker symbol,
  restrict both to inputs without it: `[~$"MARKER"] .o. Layer`. Otherwise inputs
  containing the marker make the test fail for no real reason.

## 6. Size control

RHVoice FSTs take about 8 bytes per arc and allow at most 65,535 symbols. Measured
costs:

- **Insertion guard vs substitution guard.** `[..] -> M || Stems _ z Tails` cost
  5 MB of network. The same guard as a substitution, `z -> M || Stems c _ Tails`
  followed by `M -> z`, cost nothing.
- **Mark/spell/unmark trios** for letter runs cost 3 MB. One longest-match rule of
  the shape `[[C+ %-] .o. [Names @->] .o. [%- -> 0]] @-> || .#. _` was cheaper.
- **Right contexts with Kleene stars** such as `[\LowerLetter]*` are expensive
  (+2 MB for a single rule). Contiguous contexts or phone lookahead (`"/ih0"`) are
  cheap.
- **Anchor one-word exceptions**: `{abc} _ p r` cost +0.3 MB, `.#. {abc} _ p r` about
  nothing. An unconditional `@->` for one word cost about 0.9 MB in a big network.
- **Lookahead markers in front of a per-word pipeline** blew one g2p up from
  7 to 20 MB. The Kleene loop `[Part [B Part]*]` itself cost nothing. Make the
  boundary deterministic (the hyphen itself), give fixed forms such as enclitics
  their own small transducer with `.P.`, and detect suffixes at the stress step
  instead of inserting a marker. The g2p shrank from 3.77 to 2.49 MB.
- **Parallel `,,` rules with complex contexts**, or long left contexts like
  `WP ?* Stem`, can explode (20 → 35+ MB, or out of memory). Compose sequential
  rules instead.
- **A `?` right context** in a small pg2p rule cost 0.55 MB, and an 11-symbol
  deletion with a right context cost 20 KB, where a single marker plus an
  unconditional spell-out cost almost nothing.
- **"Keep only the first/last accent mark" rules** double or treble a g2p.
- **gg2g homograph networks** cost roughly states × alphabet. Limit generated target
  lists to forms a corpus actually attests (6,078 → 1,950 pairs, 1 MB → 0.3 MB).

## 7. Very large word-symbol alphabets

foma 0.9.18 segfaults while compiling a rule cascade once one network's alphabet
passes roughly 35,000 multichar word symbols (seen in a gg2g part). Keep
generated word lists under frequency thresholds, split the rules over several
parts of a multi-part FST, and prefer a positive "take it back" rule to a
`\BigList` complement.

## 8. Where a fix belongs (layering)

Keep g2p maintainable. Put each fix in the layer that owns it:

- productive rule families: g2p,
- compound stems: a compound-parts list (if the module has a compound splitter),
- single words: the lexicon or exceptions files,
- devoicing, assimilation, cross-word effects: pg2p,
- homographs that depend on the neighbouring word: gg2g.

Do not dump word lists into g2p. It bloats the network and nobody can review it.
