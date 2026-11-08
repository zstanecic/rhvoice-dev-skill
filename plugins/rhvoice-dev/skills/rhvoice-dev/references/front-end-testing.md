# Testing front-end (language module) changes

The front end is cheap to test: transcribing tens of thousands of words takes
seconds, while a g2p build takes many minutes. So build once, then test a lot.

## Contents
1. Old vs new sweeps
2. Inflection / case matrices
3. Finding dropped tokens (numbers, tokenizer)
4. Homograph and stress work against gold data
5. A/B synthesis
6. Checking a voice's training prompts against a new front end

## 1. Old vs new sweeps

- **Get a real "old" side.** Use `git show HEAD:data/languages/X/<f>.fst` or a
  `git archive` snapshot of the language repo. Point the tools at it: swap the
  FSTs in `data/languages/X/` and back again, and check with `sha256sum -c`, or use
  a separate `RHVOICE_DATA_PATH` for `RHVoice-test`. One sweep silently took its
  "old" readings from the new network because the new FST had already been
  installed.
- **Input**: one `<s>` per word or sentence in an SSML file. Leave out empty words,
  since an empty `<s>` produces no line. Speed is about 30k words in 4 s, so
  million-word sweeps are fine (a few minutes per side).
- **Word list sources**: the module's test fixtures, the lexicon, the prompts of
  every voice corpus you have, and a web corpus (e.g. Leipzig Wortschatz
  `<lang>_web_<year>_100K`). Generate the list *after* adding any new test words.
- **Rule**: every changed line must be explained by the change. Report counts
  (changed words, changed phones, pause count) and show a sample.
- Keep the helper scripts in the repo or a tools directory, not only in a
  temporary scratch area. Otherwise they have to be rewritten for every change.

## 2. Inflection / case matrices

For agglutinative or inflecting languages, a rule that is right for the dictionary
form is often wrong for some suffix, prefix or capitalisation. Before calling a
rule done, generate:

- stem × all case endings × plural × possessive,
- × prefixes (verbal or nominal),
- × Title case and UPPER case,
- × hyphenated compounds,

and check each form against an expectation. Example for compounds:
`reading(left part) ++ reading(free-standing right part)`. Use a strict
phone-by-phone comparison, and allow only differences you can name (e.g. voicing
before an obstruent, nasal place before a consonant, vowel length). In one
module, the first drafts of a family of surname rules missed vowel-initial
suffixes, a compound-seam guard and a suffix-harmony conflict. Only the matrix
found them.

## 3. Finding dropped tokens

**Symptom**: words vanish from speech with no error. For example, in "Thursday
05 November 2026" the words "05 November" were lost.

**Cause**: the tokenizer joined a shape (leading-zero day + month, number +
unit, number + noun) into a `num` token that `numbers.fst` cannot read. The core
then writes nothing for it (see `decode_as_number` in language.cpp).

**Harness**: compile a small C++ program against the local `libRHVoice_core` that
builds the utterance for each sentence, walks the `TokStructure` relation, and
flags every non-`sym` subtoken without children. Run it over a large corpus with
digits, plus a synthetic corpus of number shapes (dates, times, ranges, amounts,
fractions, units, ordinals).

Typical classes to look for:

- leading zeros joined to nouns or units,
- `number + any word` joins that should only take an adjective,
- 0-led fractions,
- dot clocks,
- range chains (`3 – 2 – 1`),
- unit signs without a number (`°C`, `%`),
- invalid dates (`32. 12.`).

Ask a native speaker about format conventions before "fixing" them. In some
languages, for example, a space is *not* a thousands separator.

## 4. Homograph and stress work against gold data

- **Frequency defaults** usually give the biggest win. Picking the most frequent
  reading per form from a tagged frequency model beat a rule-based disambiguator
  (in one module: 83.4% → 89.0% of ambiguous tokens on hand-accented text). Add a few
  lexicalised context rules (gg2g) after that, but do not attempt a general
  grammar in an unweighted FST.
- **Checking stress**: transcribe the plain text and the same text with combining
  accent marks, split words on the `#` and `pau` tokens (not on the `#` inside
  `a1#3`), and compare the stressed-syllable index and tone per word.
- **Context-free phone tiers** in a corpus's TextGrids can check single-reading
  words but cannot judge homographs. A long-vowel mark in a TextGrid may also
  mark pretonic vowels, so it is not a reliable stress signal.
- **Lemma matching traps** in generated preferences: a reflexive infinitive and a
  noun with the same spelling got matched to each other, and the frequency
  preference then picked the wrong one. Scan the model rows of the affected tag
  class when a default looks odd.

## 5. A/B synthesis

- Synthesize the same sentences with the old and the new data using
  `RHVoice-test -p <Voice name>` and `RHVOICE_DATA_PATH`. Compare WAV size and
  SHA-256.
- Control sentences without the changed contexts must be byte-identical. If they
  are not, something else changed.
- For vowel-length or spelling-duration work, synthesize single spelled letters and
  measure the speech span. In one check the short/long ratio was 1.5–2.2.

## 6. Checking a voice's training prompts against a new front end

A voice is trained on labels made by the front end of its time. After language
changes, transcribe the prompts with the old and the new data and count the
changed labels per prompt. Many changes mean the voice should be relabelled and
retrained to stay consistent. vbu reads only `xml:id` from the prompt SSML, and
the labels come from RHVoice itself, so `-b ssil` transcription shows what
training will see.
