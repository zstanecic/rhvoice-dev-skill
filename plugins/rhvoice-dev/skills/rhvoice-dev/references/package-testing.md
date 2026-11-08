# Testing RHVoice packages and the Speech Dispatcher module without root

## Contents
1. Before building packages
2. Rootless rootfs from container images
3. Upgrade tests
4. Release checks
5. Testing the Speech Dispatcher module (`sd_rhvoice`)
6. Shell traps

## 1. Before building packages

- Run `git status` in the main repo **and in every nested data repo**. Packaging
  scripts often pack everything under a data directory (recursive glob), including
  untracked files. A voice inventory that globs `data/voices/*/voice.info` turns an
  untracked, unreleased trained voice into a new public package without warning.
  Ask the user about anything untracked under `data/voices/`, and move held-back
  voices out of the tree for the build.
- The engine loads only `voice.data` and `bpf.txt` from a voice. Untracked leftover
  old-format HTS files only bloat the package. Tracked ones in upstream voice
  repos are normal.
- A refresh ships whatever is in `data/`. Check `git log` of every language and
  voice since the last release and name each change in the release notes.
- `extra_utt_types` in voice.info had no runtime effect in the version checked:
  nothing calls `supports_utt_type()`, and labels always carry utt_type.

## 2. Rootless rootfs from container images

You can test DEB/RPM packages on Debian, Ubuntu and Fedora without root, Docker
or VMs:

1. Get the image's root filesystem layer, e.g. the gzipped OCI layer tarball of
   `debian:13`, `ubuntu:24.04`, `registry.fedoraproject.org/fedora:44`.
2. Extract it in a user namespace:

   ```
   unshare --user --map-root-user tar -xzf <layer> -C <rootfs> --no-same-owner
   ```

3. Write `<rootfs>/etc/resolv.conf` (use the host's resolver; on WSL it is the
   address in the host's `/etc/resolv.conf`). Create mount points (`proc`, `dev`,
   a `src` bind target for your tree).
4. Enter the rootfs with `unshare --user --map-root-user --mount` plus `chroot`,
   bind-mounting `/proc`, `/dev` and the source tree.
5. A single-UID user namespace makes package managers' `chown` to other users
   fail. An `LD_PRELOAD` shim that turns `chown`/`fchownat`/`lchown` into no-ops
   that return success solves it.
6. apt needs `-o APT::Sandbox::User=root`. Use `--no-install-recommends` to keep
   the rootfs small.
7. Install the runtime you want to test, e.g. `speech-dispatcher
   python3-speechd pulseaudio pulseaudio-utils python3` (Fedora also needs
   `polkit`).

Each rootfs takes 1.7–2.3 GB and about 2–3 minutes to provision, and several
can be provisioned in parallel.

**Cleanup**: Fedora trees contain mode-000 files that plain `rm` cannot delete:

```
unshare --user --map-root-user bash -c 'chmod -R u+rwX X; rm -rf X'
```

Do not `cp -a` a directory that contains rootfs snapshots into a backup by
accident. It copies GBs and leaves undeletable files.

## 3. Upgrade tests

1. Install the **previous** release in each rootfs.
2. Snapshot the rootfs you will also use for the repository upgrade test, *before*
   the local upgrade.
3. Upgrade with the new set:
   - `apt-get install -y -o APT::Sandbox::User=root ./deb/*.deb; dpkg --audit`
   - `dnf install -y ./rpm/*.rpm; dnf check`
4. Run your runtime validation (voices load, every language speaks) in each
   rootfs. These can run in parallel.
5. After publishing to an apt/dnf repository, run `apt upgrade` / `dnf upgrade`
   from the published repo in the snapshots. New packages that arrive as
   dependencies of a meta-package should be installed, and nothing should be kept
   back.
6. Compare the published files with the local set by SHA-256.

## 4. Release checks

- Synthesize test sentences with `RHVoice-test -p <Voice name>` on the old release
  (snapshot) and the new one (upgraded), on every distribution.
- Sentences without changed contexts must give **byte-identical WAVs** across
  releases and across Debian, Fedora and repo-installed packages.
- A missing voice is not detected by exit code: `-p <missing>` uses another voice
  and exits 0. Check the voice list or the output instead.

## 5. Testing the Speech Dispatcher module (`sd_rhvoice`)

**Through speechd** (inside a rootfs, without touching its packages):

- Start PulseAudio with a null sink, then run
  `speech-dispatcher -s -t 0 -S $XDG_RUNTIME_DIR/speech-dispatcher/speechd.sock
  -m <dir with the new sd_rhvoice>`.
- speechd sets the module's log level to 0, so the module log is empty. Read
  `speech-dispatcher.log` (`-l 5`) to see what was sent.
- With `SSML_MODE on`, speechd passes the client's markup through and adds its own
  `__spd_N` marks. Some speechd versions re-escape a bare `&` to `&amp;` in SSML
  mode, even with `SymbolsPreproc "no"`. Some module bugs cannot be triggered
  through speechd for this reason.
- A rootfs shares one `/tmp`, so never run two speechd tests in the same rootfs
  at once. Different rootfs can run in parallel.
- Probe scripts: plain messages send no index marks, so allow a long stall timeout
  (30 s or more) for anything longer than a sentence. Let each message finish, or
  the next one may take a leftover END as its own. A test with many marks needs
  the long default wait (90 s); shorter waits cascade into false failures.

**Driving the module directly** (no speechd; reproduces raw input):

```
INIT                                   → 299
AUDIO  audio_output_method=pulse
       audio_pulse_server=  .          → 203
SET    synthesis_voice=<Name>  .       → 203
SPEAK  <text lines>  .                 → 202 … 200, then events 701 / 700-<mark> / 702
```

Use a time limit per message (e.g. 20 s). Kill and restart a stuck module.
Malformed SSML used to hang the module: a parser exception escaped the synthesis
thread after BEGIN, so END never came. Test fragments without `<speak>`, a stray
`&`, numeric references, and `<break>` inside text.

## 6. Shell traps

- `cd X && V=... && (job1) & (job2) & wait` runs the `cd` and the assignment only
  inside the first background job. Set variables on their own line, then
  `{ job1; } & { job2; } & wait`.
- A waiter like `bash -c 'until ! pgrep -f check-x; do sleep 5; done'` never ends,
  because the waiting shell's own command line contains the pattern. Wait for a
  log line instead.
- Minimal systems may lack `rsync`, `xxd` or `/usr/bin/time`. Use bash's `time`
  keyword.
