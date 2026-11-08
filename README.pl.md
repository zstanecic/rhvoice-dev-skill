# rhvoice-dev: skill Claude do pracy nad RHVoice

[English version](README.md)

`rhvoice-dev` to skill dla Claude (Claude Code, aplikacje Claude, API) z
praktyczną wiedzą o rozwijaniu modułów językowych
[RHVoice](https://github.com/RHVoice/RHVoice) i głosów HTS. Każda notatka
pochodzi z prawdziwej pracy nad modułami językowymi RHVoice i korpusami głosów.
Opisuje pułapki, które kosztowały godziny, testy, które wyłapały regresje, i
pomiary, na których opiera się każda rada.

Claude sam wczytuje skill, gdy zadanie dotyczy drzewa RHVoice, gramatyk foma,
treningu HTK/HTS albo powiązanych narzędzi. Wczytuje tylko potrzebny temat.

## Zawartość

- [SKILL.md](plugins/rhvoice-dev/skills/rhvoice-dev/SKILL.md): przegląd, zasady
  pracy, szybkie testy, skrót o `<phoneme>`.
- [foma.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/foma.md): pułapki
  foma, „G2p failed”, dowody totalności i równoważności, kontrola rozmiaru FST.
- [front-end-testing.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/front-end-testing.md):
  testowanie zmian w module językowym. Porównania stara/nowa wersja, macierze
  odmiany, gubione tokeny, synteza A/B.
- [ssml-phoneme.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/ssml-phoneme.md):
  SSML `<phoneme ph="…">` w każdym języku. Zapis, akcent, ton, pułapki.
- [voice-training.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/voice-training.md):
  trening głosu w vbu / voice-building-utils. Kontrola korpusu, duże korpusy,
  f0 i chrypka (vocal fry), VAD, wiele rdzeni, Praat 7.
- [htk-alignment-diagnosis.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/htk-alignment-diagnosis.md):
  co naprawdę powiedział lektor. Diagnoza `problematic_files` modelami samego
  głosu.
- [toolchain.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/toolchain.md):
  budowanie HTK, HTS, SPTK, speech_tools/festival z GCC 15–16 na Fedorze 44–45.
- [espeak-import.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/espeak-import.md):
  import reguł eSpeak NG do g2p w foma.
- [package-testing.md](plugins/rhvoice-dev/skills/rhvoice-dev/references/package-testing.md):
  testowanie paczek DEB/RPM i modułu Speech Dispatchera w rootfs bez roota.

Pliki są zwykłym Markdownem, więc ludzie też mogą je czytać. Treść skilla jest
po angielsku, żeby mógł z niego korzystać każdy.

## Instalacja

### Claude Code jako plugin (zalecane)

Wpisz w Claude Code:

```
/plugin marketplace add zstanecic/rhvoice-dev-skill
/plugin install rhvoice-dev@rhvoice-dev-skill
```

Aktualizacja: `/plugin marketplace update rhvoice-dev-skill`.

### Claude Code jako skill osobisty (we wszystkich projektach)

```sh
git clone https://github.com/zstanecic/rhvoice-dev-skill.git
mkdir -p ~/.claude/skills
ln -s "$PWD/rhvoice-dev-skill/plugins/rhvoice-dev/skills/rhvoice-dev" ~/.claude/skills/rhvoice-dev
```

Aktualizacja: `git pull` w sklonowanym katalogu.

### Claude Code dla jednego repozytorium (dla każdego, kto je sklonuje)

Skopiuj katalog `plugins/rhvoice-dev/skills/rhvoice-dev` do repozytorium jako
`.claude/skills/rhvoice-dev` i zrób commit.

### Aplikacje Claude (claude.ai, Desktop)

Spakuj katalog `plugins/rhvoice-dev/skills/rhvoice-dev` do pliku zip, tak żeby w
środku był `rhvoice-dev/SKILL.md`. Potem wgraj go jako własny skill w sekcji
Skills w ustawieniach.

## Użycie

Nie trzeba wywoływać skilla z nazwy. Przykłady próśb, przy których się wczyta:

- „Transkrypcja zwraca `G2p failed: <słowo>`, znajdź przyczynę.”
- „g2p.fst urósł z 20 do 30 MB po tej regule, jak go zmniejszyć?”
- „Niech RHVoice przeczyta to słowo inaczej, do testu.”
- „W problematic_files_3.txt jest 12 plików, sprawdź, co powiedział lektor.”
- „SPTK nie buduje się na Fedorze 45.”

W Claude Code można go też wywołać bezpośrednio: `/rhvoice-dev`.

## Zakres i weryfikacja

- Sprawdzone wersje: RHVoice 1.19.x, foma 0.9.18, HTK 3.4.1 z HTS 2.2/2.3,
  SPTK 3.11, Praat 6.4–7.0, Fedora 44/45, Debian 13, Ubuntu 24.04/26.04.
- To, co sprawdzono w kodzie albo uruchamiając narzędzia, jest zapisane jako
  fakt. To, co sprawdzono tylko na jednej konfiguracji, jest tak oznaczone.
- Liczby (rozmiary, czasy, procenty) to pomiary z prawdziwych modułów i
  korpusów. Traktuj je jako rząd wielkości.
- Skill nie zawiera danych osobowych, ścieżek, serwerów ani haseł.

## Współpraca

Zgłoszenia i pull requesty są mile widziane. Nowe odkrycie najprościej zgłosić
przez [formularz](https://github.com/zstanecic/rhvoice-dev-skill/issues/new?template=finding.yml).
Jeśli używasz skilla w Claude, Claude może sam zaproponować wypełnienie
formularza, gdy trafi na coś nowego i ogólnego. Nigdy niczego nie wysyła sam:
przeglądasz tekst i wysyłasz go Ty, a opiekun repozytorium sprawdza go przed
scaleniem.

Zasady:

- tylko rzeczy sprawdzone w praktyce, z wersją i sposobem sprawdzenia;
- powód i pomiar, nie sama reguła;
- ogólne sformułowania: przykład może dotyczyć jednego języka, ale rada ma
  pasować do innych;
- `SKILL.md` ma zostać krótki, a szczegóły idą do odpowiedniego pliku w
  `references/`;
- żadnych danych prywatnych: ścieżek, nazw serwerów, haseł, nazwisk, nazw
  korpusów i lektorów, niewydanej pracy.

## Licencja

[MIT](LICENSE)
