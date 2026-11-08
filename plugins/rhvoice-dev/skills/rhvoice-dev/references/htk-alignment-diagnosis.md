# Finding what the speaker really said, with the voice's own HTK models

For files in `problematic_files_N.txt` (HVite needed beam widening, "No tokens
survived"), you can test hypotheses about the actual reading in seconds, without
ASR and without listening. Use the HMMs of a finished segmentation round
(`htk/hmm/<last>/`).

## Workspace pieces you need

- `htk/words.mlf`: phones written as "words". `word_final_X` = X with an optional
  pau; `X_before_punctuation`.
- `htk/dict`, `htk/hmm/<last>/`, and `file_name_mapping.json`
  (`RHVoice_<Spk>_NNNN` → original id).
- Sanity check before comparing anything: `RHVoice-transcribe-sentences -b ssil`
  plus vbu's word_final/_before_punctuation rules must reproduce `words.mlf`
  exactly.

## 1. Symptoms in the alignment

- In `phonesN.mlf`, a run of ≥3 phones at the 5-frame minimum means the text has
  something the speaker did not say.
- A phone longer than the 99.5th percentile of its own duration means the speaker
  said more than the text.
- Per-phone scores: rerun HVite without `-o S` and compute z-scores against ~300
  random files.

## 2. Variant test

1. Replace the suspect span in `words.mlf` with a token `TOKn`.
2. Give `TOKn` several pronunciations in a copy of the dict, plus a copy of each
   with `pau`.
3. Run `HVite -a -m -o S …`. The chosen pronunciation shows in the output.

For an omission, keep one neighbouring phone in every variant, because HTK cannot
have an empty pronunciation.

## 3. Margins, with controls

Force each variant alone and compare the `Ac=` totals. **Controls are essential.**
On clean files a shortened form lost or won by ≤ 70. Real misreadings won by
370–3,700. Run the same variants on a few files where you know the text is right.

## 4. Confirm with a phone loop

Variant tests favour shorter forms in fast speech, so confirm with free phone
recognition over the span:

```
$ph = <all phones | SILENCE>;
( SILENCE words-before < $ph > words-after SILENCE )
```

Build it with HParse and run `HVite -w net …`. Retry with `-t 0` when the beam gives
no output. Example: the variant test preferred a form with one syllable dropped
(+370), but the loop recognised the full word, so the prompt stayed.

## Pitfalls

- `pau` is not a dict word; use `SILENCE` in grammars.
- Use the HMMs of the same training run that produced the problematic files.
  Models from another run, or labels from a different front-end revision, make
  the margins meaningless.

## After the diagnosis

Fix the prompt (text, or a `<phoneme>` override, see
[ssml-phoneme.md](ssml-phoneme.md)), then realign old vs new labels with the
original `-t 250 150 1000`. Report the per-file score change and the change in
beam-250 failures. In one round all 47 changed files scored better (mean
+0.43/frame) and failures dropped from 9 to 0. Changed labels mean training must
be rerun from the start.
