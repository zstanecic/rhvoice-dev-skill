# Building the voice-building toolchain with modern compilers

HTK 3.4.1, HTS 2.2/2.3, hts_engine, SPTK 3.11, speech_tools/festival and Praat on
current distributions. Every fix below was verified on Fedora 44 (GCC 15/16) and
Fedora 45 (GCC 16.2, glibc 2.44, dnf5 5.4.6), in containers and by a user on real
machines. Debian/Ubuntu users meet the same compiler issues.

## Contents
1. dnf multilib traps (32-bit HTK)
2. HTK and HTS
3. SPTK 3.11
4. speech_tools and festival
5. Downloads
6. Praat

## 1. dnf multilib traps

HTK is built 32-bit (`-m32`), so you need `.i686` devel packages next to the
x86_64 ones.

- **dnf5 5.4.6 installs ONLY the i686 package** when a name is given arch-less
  next to `name.i686` in the same transaction: `dnf install libX11-devel
  libX11-devel.i686` or `libatomic libatomic.i686`. dnf5 5.4.5 installed both,
  which hides the problem on older systems. **Rule: every package you also install
  as `.i686` must be written with `.x86_64`**:

  ```
  dnf install libX11-devel.x86_64 libX11-devel.i686 libatomic.x86_64 libatomic.i686 …
  ```

- Symptom of a missing x86_64 libatomic: `C++ compiler cannot create executables`.
  config.log shows that `/usr/lib64/libatomic.so.1.2.0` cannot be found.
- GCC 16 adds `-latomic_asneeded` to every link, so 32-bit HTK needs
  `libatomic.i686`.
- With only `libX11-devel.i686` installed, autoconf's AC_PATH_X finds
  `/usr/lib/libX11.so` and sets `X_LIBS=-L/usr/lib`. 64-bit X programs then fail
  with `skipping incompatible /usr/lib/libX11.so` / `cannot find -lX11`.

## 2. HTK and HTS

- **HTS 2.3 patch configure bug**: `AC_CANONICAL_HOST` runs only inside the Fortran
  branch, so without gfortran it drops `-ansi`, ARCH etc. The wiki's
  `CFLAGS="-DARCH=linux"` fails on GCC ≥ 14 either way. Fix: install
  `gcc-gfortran` and configure **without** CFLAGS. HTS 2.2 is fine without
  Fortran.
- HSLab needs the x86_64 libX11-devel (see above), or build with X disabled.

## 3. SPTK 3.11

Four failures, in the order they appear:

1. configure: `Cannot find csh`. Install `tcsh`.
2. make: GCC ≥ 15 defaults to C23, where `FILE *getfp();` means "no arguments".
   Errors appear in `bin/dct/dct.c`, `ulaw.c`, `bin/pitch/snack/{jkGetF0,sigproc}.c`.
   Fix: `CFLAGS="-g -O2 -std=gnu17"`.
3. make: without `gcc-c++`, configure still passes (it silently sets `CXX=g++`)
   and REAPER in `bin/pitch` fails with Error 127 `g++: not found`. Install
   `gcc-c++`.
4. link: `multiple definition of bbm` in psgr (GCC ≥ 10 defaults to
   `-fno-common`). Fix: add `-fcommon` to CFLAGS, or make the variable `extern` in
   `psgr.h` and define it once in `psgr.c`.

With all fixes, 138 programs are installed, or 139 with `xgr` when x86_64
libX11-devel is present. `--without-x` skips xgr.

## 4. speech_tools and festival (GitHub festvox)

```sh
cd speech_tools
./configure CFLAGS="-g -O2 -std=gnu17" CXXFLAGS="-g -O2 -std=gnu++17"
make && make test        # ends with "Test OK"
cd ../festival && ./configure && make && make default_voices
```

- C: the C23 default breaks `editline.c` (`conflicting types for 'getenv'`,
  `too many arguments to function 'tgetent'`), so pass `-std=gnu17`.
- C++: GCC 16's C++20 default makes `relation` ambiguous in `EST_Utterance.cc`, so
  pass `-std=gnu++17`.
- Repeat `-g -O2`: setting CFLAGS replaces configure's default.
- Festival inherits the flags through `speech_tools/config/user_flags.mak`. The two
  directories must be siblings named exactly `speech_tools` and `festival`.
- **Do not use `make -j`**: sub-makes write to the same `lib/*.a` archives
  (`file truncated`, missing `libeststring.a`).
- `make default_voices` ignores wget failures and exits 0. Check that
  `festival/packed/` holds `festvox_kallpc16k.tar.gz`, `festlex_CMU.tar.gz` and
  `festlex_POSLEX.tar.gz`. Without them `text2wave` writes empty WAVs.
- There is no install step. `bin/` holds absolute symlinks and the lib path is
  compiled in, so build in the final location.

## 5. Downloads

`kumisystems.dl.sourceforge.net` (still in some instructions) is dead. Use
`curl -LO https://downloads.sourceforge.net/…`.

## 6. Praat

Fedora has no praat package. Use the upstream static `praat…_linux-x64…barren`
build. For Praat 7 and `--FULL-TRUST`, see
[voice-training.md](voice-training.md#7-praat-7).
