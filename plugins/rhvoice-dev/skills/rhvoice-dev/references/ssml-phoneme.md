# SSML `<phoneme>` overrides in RHVoice, for any language

Use them to hear a reading the language module cannot produce yet, to pin one
word in a training prompt, or to test whether a different transcription would fix
an alignment or a pronunciation bug before writing rules.

## How RHVoice handles it (from the code)

- Handler: `phoneme_handler` in `src/include/core/ssml.hpp`. It accepts
  `alphabet=""` (absent) or `alphabet="x-RHVoice"`. **With any other alphabet
  (`ipa`, `x-sampa`) the word is silently dropped**: the content is not read
  either (verified: `a <phoneme alphabet="ipa" ph="kæt">cat</phoneme> and a dog`
  transcribes as "a and a dog").
- `<phoneme ph="...">text</phoneme>` and `<phoneme ph="..."/>` behave the same: the
  element content is ignored, and only `ph` is used.
- `ph` is split on whitespace (`language::set_user_phones` in `language.cpp`). Each
  piece becomes one segment of the word's transcription. The word skips the
  lexicon and g2p, **but `pg2p.fst` still runs over the whole phrase**, so
  cross-word assimilation can still change it.
- Then `syl.fst` syllabifies the word as usual (`language::syllabify`):
  - a segment ending in `1` is a stressed vowel and one ending in `0` an unstressed
    vowel; the digit is stripped from the phone name,
  - a separate token `1`…`9` right after a vowel sets that syllable's lexical tone
    (`lex_tone`),
  - a leading `/` is stripped (internal notation of some modules; do not use it).

## Notation per language

The symbols are the language's own phone names from
`data/languages/<Language>/phonemes.xml`, never IPA. Find them by transcribing a
similar word with `RHVoice-transcribe-sentences -v` and translating the `-v`
output back:

- **Stress languages**: put `1` on the stressed vowel. Without it the word gets no
  stress at all. English is used as the example here:

  ```
  <phoneme ph="r eh1 k er0 d">record</phoneme>
  ```

  `-v` gives `r eh1 . k er d`.

- **Tonal / pitch-accent languages**: unstressed vowels take `0`, and the stressed
  vowel takes `1` followed by a separate tone digit. `-v` prints this as `a1#2`;
  write it as `a1 2`. Schematic example (use your language's phone names):

  ```
  <phoneme ph="t a0 l a1 2 m">…</phoneme>
  ```

- **Unknown symbol** (not in `phonemes.xml`): `Unknown phoneme: xx`. The transcriber
  exits 255 and writes nothing after that point, and in synthesis the sentence is
  lost. Always check new `ph` strings with the transcriber first.

## Pitfalls

- **Punctuation right after `</phoneme>` does not attach to the token.**
  `…</phoneme>, and` loses the comma's pause (the `-v` output shows `#` instead of
  `pau`). Put `<break strength="medium"/>` before the comma. It reproduces the
  comma's pause exactly (verified in more than one language):

  ```
  a <phoneme ph="k ae1 t"/><break strength="medium"/>, and a dog
  ```

- Writing `/p /a /m`-style phones (slash prefix) gave a word no syllable split,
  stress or tone. Use plain phone names.
- Always diff the old and new `-v` transcription after editing prompts: lines
  changed, pause count.

## Use in voice training

vbu reads only `xml:id` from the prompt SSML, and the labels come from RHVoice. So
a `<phoneme>` in a training prompt changes the labels the voice is trained on.
This is the cleanest way to record that the speaker said something different from
the text (dropped endings, a one-syllable acronym, a misread form). Keep a backup
of the prompt file, and rerun training from the start after changing labels. To
choose the right `ph`, see
[htk-alignment-diagnosis.md](htk-alignment-diagnosis.md).
