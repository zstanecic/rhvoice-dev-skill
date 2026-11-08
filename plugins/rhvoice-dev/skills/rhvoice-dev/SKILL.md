---
name: rhvoice-dev
description: Field-tested know-how for developing RHVoice languages and voices. Covers foma grammars (tok, numbers, g2p, gg2g, pg2p) and their silent traps, FST size control, testing front-end changes with RHVoice-transcribe-sentences, SSML <phoneme ph="..."> overrides to debug or pin the pronunciation of a word in any language, HTS/HTK voice training with voice-building-utils/vbu (corpus preparation, problematic_files, alignment diagnosis, f0 range, multicore, VAD silence init), building HTK/HTS/SPTK/speech_tools with modern GCC on Fedora, Praat 7 --FULL-TRUST, importing eSpeak NG rules, and testing packages or the Speech Dispatcher module without root. Use it whenever the user works in an RHVoice tree (src/scripts/<Language>, data/languages, data/voices, training.cfg) or mentions foma, .fst, g2p, "G2p failed", HTK, HTS, HVite, HERest, "No tokens survived", voice-building-utils, eSpeak rules or RHVoice packaging, even if they do not say "RHVoice".
---

# RHVoice language and voice development

Notes from many months of work on RHVoice language modules and HTS voices. Every
item was hit in practice. Where something was verified only on one setup, the
reference file says so. Read this file first, then open only the reference you need.

## Orientation

- **Language sources** are in `src/scripts/<Language>/`: foma grammars, generator
  scripts, tests such as `checktrans`. **Compiled data** is in `data/languages/<Language>/`:
  `*.fst`, `phonemes.xml`, `labelling.xml`, and `language.info` with the `revision`.
  **Voices** are in `data/voices/<name>/` (`voice.info`, `voice.data`, `bpf.txt`). In
  many trees these directories are separate git repositories (submodules or
  nested repos). Run `git -C <dir> status` on each, and after committing inside one,
  commit the bumped pointer in the parent repo.
- **Text pipeline**, roughly: `tok.fst` (tokens and their types) → norm/downcase →
  `numbers.fst`, `spell.fst`, `key.fst` for non-words → `gpos.fst` → the language's
  lexicon/`g2p.fst` (and `gg2g.fst` homograph rules where present) → `pg2p.fst`
  (cross-word phonology over the whole phrase) → `syl.fst` (syllables, stress,
  tone) → HTS labels. Check `src/core/language.cpp` and the language's own `.cpp`
  for the exact order in your version.
- Runtime `.fst` files use RHVoice's own binary format, converted from foma's ATT
  output. You cannot `flookup` them. Test through the RHVoice tools, or compile
  the `.foma` source in foma (and mind the tokenisation trap in
  [references/foma.md](references/foma.md)).

## Working principles

1. **Fix language problems in language data, not in the C++ core.** A core change
   affects every language and needs the maintainers' agreement. Reading core code
   to learn which features exist is fine. If only a core change would really help,
   explain why and ask before making it.
2. **Measure every change.** Transcribe a large word list or corpus with the old and
   the new data and explain every difference. Keep a pristine old copy (`git show
   HEAD:<file>`, `git archive`) as the "old" side. Install the new FST only after all
   old-vs-new readings are done, otherwise you compare new against new without
   noticing.
3. **Long builds:** a big g2p network can take 10–25 minutes to compile. Run it in
   the background, and do not edit any sourced `.foma` file while foma is compiling
   (see the foma reference for why).
4. **Respect the machine.** Voice corpora are tens of GB, and training, ASR or
   synthesis over a corpus runs for hours. Ask before starting such runs.
   Transcribing a corpus's prompts takes seconds and is fine.
5. **Watch FST size.** It costs load time and package size, and some rule shapes
   multiply it. See [references/foma.md](references/foma.md).
6. **Don't trust exit codes alone.** `RHVoice-test -p <unknown name>` silently uses
   another voice and exits 0. The transcriber aborts the whole file at the first
   unknown phoneme. A tokenizer path that `numbers.fst` cannot read makes RHVoice
   drop those words without any error.

## Fast checks

`RHVoice-transcribe-sentences [-v] [-b MARK] in.ssml out.txt` writes one line per
`<s>` (or per sentence). The `-v` output reads like this:

- `pau` is a pause,
- `ao1`: a `1` after a vowel marks a stressed syllable,
- `#2`: the syllable's lexical tone (in tonal or pitch-accent languages),
- ` . ` separates syllables,
- `(det)` is the gpos of a non-content word,
- `#` separates words (`-b MARK` replaces it).

```
pau ax (det) # k ae1 t # ae1 n d (cc) # ax (det) # d ao1 g pau
```

Traps:

- Write to a real file. Capturing through `/dev/stdout` truncates it; pipe through
  `| cat` if you must.
- Errors that stop the run (exit 255): `G2p failed: <word>`, `Unknown phoneme: <x>`,
  `Tokenization failed`.
- An empty `<s></s>` gives no output line, so line counts stop matching.
- `RHVoice-test -p <Name>` takes the voice name from `voice.info`, not the directory
  name. `RHVoice-test` honours `RHVOICE_DATA_PATH` (a directory with `languages/` and
  `voices/`), which is handy for old-vs-new synthesis. For a sentence without new
  contexts, old and new WAVs should be byte-identical.

## SSML phoneme overrides, for any language

`<phoneme ph="k ae1 t">cat</phoneme>` (or self-closing `<phoneme ph="..."/>`) puts
the given phones straight into the word's transcription:

- Phones are the language's own symbols from `phonemes.xml`, separated by spaces.
- A `1` after a vowel marks stress.
- A separate digit token after the vowel sets the tone where the language has
  tones (`a1 2`).
- `syl.fst` still syllabifies the word and `pg2p` still applies cross-word rules.
- A comma right after `</phoneme>` loses its pause. Add `<break strength="medium"/>`
  before the comma.

This is the quickest way to hear or align a reading the module cannot produce
yet, and to fix single prompts in a training corpus. Full rules, the notation for
each kind of language, and pitfalls are in
[references/ssml-phoneme.md](references/ssml-phoneme.md).

## Which reference to read

- Writing or debugging foma rules, a network that blows up in size, `G2p failed`,
  `Tokenization failed`, proving two grammars are equivalent:
  [references/foma.md](references/foma.md)
- Testing a front-end change (sweeps, inflection matrices, words dropped from
  numbers or the tokenizer, A/B synthesis):
  [references/front-end-testing.md](references/front-end-testing.md)
- Pinning or debugging a pronunciation with `<phoneme>`:
  [references/ssml-phoneme.md](references/ssml-phoneme.md)
- Preparing a corpus and training a voice with vbu/voice-building-utils, choosing
  the f0 range, very large corpora, multicore, VAD:
  [references/voice-training.md](references/voice-training.md)
- Finding what a speaker really said in `problematic_files` (HVite "No tokens
  survived"), using the voice's own HMMs:
  [references/htk-alignment-diagnosis.md](references/htk-alignment-diagnosis.md)
- Building HTK, HTS, SPTK, speech_tools or festival with GCC 14–16 / Fedora 44–45,
  and Praat 7: [references/toolchain.md](references/toolchain.md)
- Importing eSpeak NG rules or exceptions into a foma g2p:
  [references/espeak-import.md](references/espeak-import.md)
- Testing DEB/RPM packages or the Speech Dispatcher module in rootless rootfs:
  [references/package-testing.md](references/package-testing.md)

## Sharing a new finding

People who use this skill can improve it. Sometimes you and the user hit something
**new, verified and generic**: a trap, a fix, a method or a measurement that would
help with other languages or setups, and that the references here do not cover.
When that happens, mention it once at the end of the task and offer to prepare a
contribution:

- Draft the text for the issue form at
  https://github.com/zstanecic/rhvoice-dev-skill/issues/new?template=finding.yml
  (topic, finding, versions, how it was verified, measurement). Or draft a pull
  request against the matching file in `references/`.
- Only the user decides whether to send it. Never open issues, push, or post
  anything yourself.
- Before showing the draft, remove anything private: paths, host and server
  names, credentials, people's names, corpus or speaker names, and unreleased
  language or voice work. Ask the user what may be public. When unsure, use
  schematic examples like the ones in these references.
- Skip one-off fixes, unverified guesses and project-specific state.

If the user maintains this skill (the skill directory is a git clone they own),
edit the matching reference file there instead, with their agreement.
