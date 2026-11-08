# rhvoice-dev: a Claude skill for RHVoice development

[Polska wersja](README.pl.md)

`rhvoice-dev` is a skill for Claude (Claude Code, the Claude apps, or the API).
It holds practical knowledge for working on
[RHVoice](https://github.com/RHVoice/RHVoice) language modules and HTS voices.
Every note comes from real work on RHVoice language modules and voice corpora:
the traps
that cost hours, the checks that caught regressions, and the measurements behind
each recommendation.

Claude loads the skill on its own when a task involves an RHVoice tree, foma
grammars, HTK/HTS training or the related tools. It reads only the topic it
needs.

## What is inside

| Topic | File |
| --- | --- |
| Overview, working principles, quick checks, `<phoneme>` summary | [SKILL.md](plugins/rhvoice-dev/skills/rhvoice-dev/SKILL.md) |
| foma traps, "G2p failed", totality/equivalence proofs, FST size control | [foma.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/foma.md) |
| Testing front-end changes: old-vs-new sweeps, inflection matrices, dropped tokens, A/B synthesis | [front-end-testing.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/front-end-testing.md) |
| SSML `<phoneme ph="…">` in any language: notation, stress and tone, pitfalls | [ssml-phoneme.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/ssml-phoneme.md) |
| Voice training with vbu / voice-building-utils: corpus checks, large corpora, f0 and vocal fry, VAD, multicore, Praat 7 | [voice-training.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/voice-training.md) |
| What did the speaker really say? Diagnosing `problematic_files` with the voice's own HMMs | [htk-alignment-diagnosis.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/htk-alignment-diagnosis.md) |
| Building HTK, HTS, SPTK, speech_tools/festival with GCC 15–16 on Fedora 44–45 | [toolchain.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/toolchain.md) |
| Importing eSpeak NG rules into a foma g2p | [espeak-import.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/espeak-import.md) |
| Testing DEB/RPM packages and the Speech Dispatcher module in rootless rootfs | [package-testing.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/package-testing.md) |

The reference files are plain Markdown, so people can read them too.

## Installation

### Claude Code, as a plugin (recommended)

Type these two commands in Claude Code:

```
/plugin marketplace add zstanecic/rhvoice-dev-skill
/plugin install rhvoice-dev@rhvoice-dev-skill
```

To get updates later: `/plugin marketplace update rhvoice-dev-skill`.

### Claude Code, as a personal skill (all your projects)

```sh
git clone https://github.com/zstanecic/rhvoice-dev-skill.git
mkdir -p ~/.claude/skills
ln -s "$PWD/rhvoice-dev-skill/plugins/rhvoice-dev/skills/rhvoice-dev" ~/.claude/skills/rhvoice-dev
```

`git pull` in the clone updates the skill.

### Claude Code, for one repository (everyone who clones it)

Copy the folder `plugins/rhvoice-dev/skills/rhvoice-dev` into the repository as
`.claude/skills/rhvoice-dev` and commit it.

### Claude apps (claude.ai, Desktop)

Zip the folder `plugins/rhvoice-dev/skills/rhvoice-dev`, so that the zip contains
`rhvoice-dev/SKILL.md`. Then upload it as a custom skill in the Skills section of
the settings.

## Using it

You do not need to call the skill by name. Examples of requests that load it:

- "The transcriber says `G2p failed: <word>`, find out why."
- "My g2p.fst grew from 20 to 30 MB after this rule, how do I make it smaller?"
- "Make RHVoice read this word as /ˈrɛkərd/ for a test."
- "problematic_files_3.txt has 12 files, check what the speaker said."
- "SPTK does not build on Fedora 45."
- "Import the new eSpeak NG rules for my language."

In Claude Code you can also call it directly with `/rhvoice-dev`.

## Scope and verification

- Versions seen: RHVoice 1.19.x, foma 0.9.18, HTK 3.4.1 with HTS 2.2/2.3,
  SPTK 3.11, Praat 6.4–7.0, Fedora 44/45, Debian 13, Ubuntu 24.04/26.04.
- Claims checked in the code or by running the tools are written as facts.
  Claims verified only on one setup say so (e.g. the Praat 7 behaviour comes from
  its source code, and the detection fix was tested on 6.4.67).
- Numbers (sizes, timings, percentages) are measurements from real modules and
  corpora. Use them as orders of magnitude.
- The skill contains no personal data, paths, servers or credentials.

## Contributing

Issues and pull requests are welcome. To share a new finding, use the
[finding form](https://github.com/zstanecic/rhvoice-dev-skill/issues/new?template=finding.yml).
If you use the skill in Claude, Claude may offer to draft the form for you when
it hits something new and generic. It never sends anything itself: you review
the text and submit it, and the maintainer reviews it before merging.

Please keep the bar the notes were written to:

- only things you have hit or verified, with the version and how you checked;
- the reason and the measurement, not just the rule;
- generic wording: examples may name a language, but the advice should hold for
  others;
- `SKILL.md` stays short, and details go into the matching file under
  `references/`;
- no private data: paths, server names, credentials, people's names, corpus or
  speaker names, unreleased work.

## License

[MIT](LICENSE)
