---
name: prepare-openjdk-security-update-packages
description: Prepare openjdk-NN packages for Ubuntu LTS releases
---

# Prepare openjdk-NN packages for Ubuntu LTS releases

## Basics
The skill is only automation, which could also be achieved by scripting, to create package updates for a given OpenJDK-NN package in devel, for the currently supported Ubuntu releases.

### Arguments to this skill
1. The openjdk package: which could be openjdk-8, -lts, -17, -21, -25 and the latest interim release.
2. Revision number: default empty. If you are running this skill for a given package for the Nth time, pass N.

### Pre-requisites (no skill action needed here):
1. The user should be a part of the OpenJDK Security team on Launchpad.
2. The user should have GPG keys set up locally, and on Launchpad.
3. Each of the supported releases should have a local chroot/schroot available.
4. The user should be a member of the `sbuild` group, with `root-groups=sbuild` in the schroot configs. This gives passwordless root inside the chroots via `schroot -u root`, so no `sudo` (i.e. no password prompt) is ever needed.

### Non-interactive execution (zero human prompts)
This skill is automation: no step may wait for human input. Apply these measures for the entire run:

1. Export the following environment once, up front:
```
export GIT_TERMINAL_PROMPT=0 DEBIAN_FRONTEND=noninteractive EDITOR=true VISUAL=true
```
   - `GIT_TERMINAL_PROMPT=0`: `git ubuntu clone` / `git ubuntu export-orig` otherwise hang on a `Username for 'https://git.launchpad.net':` credential prompt.
   - `DEBIAN_FRONTEND=noninteractive`, together with `apt-get -y`: no apt/debconf questions inside the chroots.
   - `EDITOR=true VISUAL=true`: nothing ever opens an editor.
2. Redirect stdin from `/dev/null` (`</dev/null`) on long-running commands, so any hidden prompt fails fast instead of hanging.
3. Do NOT use `sudo` anywhere (see pre-requisite 4).
4. Do NOT use `dch -i` (it opens an editor). Use the non-interactive form shown in step 5.
5. Build packages unsigned (`-uc -us`), so no GPG pinentry can trigger mid-run. Signing happens once, at the end, just before the push (step 8) — the only point where a passphrase may be requested. Preset it in gpg-agent (`/usr/lib/gnupg/gpg-preset-passphrase`) if even that must be avoided.

### Support matrix
Please note the openjdk package versions and the supported Ubuntu releases:

| OpenJDK version | List of supported Ubuntu versions that need the package update | Interim release? |
| --- | --- | --- |
| openjdk-26 | 26.04 | Y | 
| openjdk-25 | 26.04, 24.04, 22.04 | N |
| openjdk-21 | 26.04, 24.04, 22.04, 20.04 | N |
| openjdk-17 | 26.04, 24.04, 22.04, 20.04, 18.04 | N |
| openjdk-8  | 26.04, 24.04, 22.04, 20.04, 18.04, 16.04 | N | 

### Ubuntu version to release name mapping
| Version | Release name | 
| --- | --- |
| 26.04 | resolute | 
| 24.04 | noble |
| 22.04 | jammy |
| 20.04 | focal |
| 18.04 | bionic |
| 16.04 | xenial |

## Steps to prepare and push package updates

### 1. Finalize Ubuntu versions list
From the openjdk-NN argument passed to the skill, and the support matrix, create a list of Ubuntu releases that we want to push the updated package to.

### 2. Ensure schroots exist
The schroots are typically named using the format <ubuntu-release-name>-<arch>. For example: `noble-amd64`.

Using the command `schroot -l`, ensure the schroots for every entry in the release list (step 1) are present locally.

### 3. Clone openjdk-NN
Clone openjdk-NN (argument to the skill) using git-ubuntu:
```
git ubuntu clone openjdk-NN
```
After the download completes `cd` into the directory:
```
cd openjdk-NN
```

Note: near the end of the clone, git may print `fatal: could not read Username for 'https://git.launchpad.net'` (it fails fast with `GIT_TERMINAL_PROMPT=0`). This is harmless — an optional ref fetch fails, but the clone itself is complete. Verify with `git status` inside the repo.

Now, the next steps should be repeated for every release in the release list, starting from 26.04 and going down.

### 4. Regenerate the debian/control file

Throw away the uncommitted changes in git.

The regeneration must run inside each release's chroot, because debian/rules uses the chroot's `lsb_release` codename to resolve release-specific conditionals.

Do this via a schroot session as root. This needs no sudo and no bind mounts; the session is overlay-backed, so chroot changes are discarded when the session ends — only the files explicitly copied back out persist.

```
SESSION=$(schroot -q -b -c <release-name>-amd64)
schroot -q -r -c $SESSION -u root --directory=/ -- sh -c 'apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -y -qq wdiff lsb-release'
schroot -q -r -c $SESSION -u root --directory=/ -- mkdir -p /openjdk-NN
tar -C $PWD -cf - debian bin $(find test/ -name TEST.ROOT) | schroot -q -r -c $SESSION -u root --directory=/ -- tar -xf - -C /openjdk-NN
schroot -q -r -c $SESSION -u root --directory=/openjdk-NN -- make -f debian/rules
schroot -q -r -c $SESSION -u root --directory=/ -- tar -cf - -C /openjdk-NN debian | tar -xf - -C $PWD
schroot -q -e -c $SESSION
```

Notes:
1. The default target of debian/rules is `debian/control`, so plain `make -f debian/rules` regenerates it (along with debian/tests/control and debian/watch).
2. `make` exits non-zero when debian/control actually changed ("debian/control did change, please restart the build..."). This is expected — the regenerated file is left in place.
3. The `test/**/TEST.ROOT` files MUST be included in the copy: debian/rules computes the jtreg dependency from `find test/ -name TEST.ROOT`. Without them, debian/tests/control silently gets a broken `jtreg8 (>= )` dependency.
4. Copying only `debian/`, `bin/` and the TEST.ROOT files (a few MB) is sufficient; never copy the full ~1 GB tree.
5. For the devel release (e.g. resolute) the regenerated control is normally identical to the one in git — an empty diff is expected there.

Dump the diff into a file named `../<openjdk-NN>-<release-name>.diff`:
```
git diff > ../<openjdk-NN>-<release-name>.diff
```

### 5. Updating the debian/changelog

Add a new entry non-interactively (`dch -i` opens an editor — do not use it):

```
dch -b -v "<prev-version-entry>~<ubuntu-version>[.<revision>]" -D <release-name>-security -u medium "Upload to Ubuntu <ubuntu-version>."
dch -a "d/control: regenerate for <release-name>"
```

The resulting entry should have the following format:

```
openjdk-NN (<prev-version-entry>~<ubuntu-version>[.<revision>]) <release-name>-security; urgency=medium

  * Upload to Ubuntu <ubuntu-version>.
  * d/control: regenerate for <release-name>
```
If the ../<openjdk-NN>-<release-name>.diff is empty DO NOT add the `d/control: regenerate for ....` message above.

Here:
1. prev-version-entry is the package version as per the previous entry in the debian/changelog
2. ubuntu-version: 26.04, 24.04 etc.
3. release-name: resolute, noble etc.
4. revision-number: passed to the skill.
5. `-b` (--force-bad-version) is required: `~<ubuntu-version>` sorts lower than the current version and `dch` aborts without it.
6. Every release's entry derives from the same base version (e.g. 25.0.4+7-1 becomes 25.0.4+7-1~26.04, 25.0.4+7-1~24.04 and 25.0.4+7-1~22.04), matching the historical per-release tags.

Don't include [.revision-number] in the default case.

Example (includes new and previous entries):
```
openjdk-21 (21.0.11+10-1~20.04.2) focal-security; urgency=medium

  * Upload to Ubuntu 20.04.

openjdk-21 (21.0.11+10-1) unstable; urgency=medium

  * OpenJDK 21.0.11 release, build 10.
    - CVEs:
      + CVE-2026-22016: 8370529: Enhance Path Factories Redux
      + CVE-2026-34282: 8374557: Enhance TLS connection handling
      + CVE-2026-22021: 8371830: Enhance certificate chain validation
      + CVE-2026-22013: 8370615: Improve Kerberos credentialing
      + CVE-2026-23865: 8379158: Update FreeType to 2.14.2
      + CVE-2026-22018: 8370986: Enhance Zip file reading
      + CVE-2026-22007: 8369575: Enhance crypto algorithm support
      + CVE-2026-34268: 8371935: Enhance key generation
  * d/rules: Check generated files only on amd64. This resolves riscv64
    ftbfs, as some architectures change with_check flag.
  * d/t/problems.csv: Fix typo in loong64 excluded tests lists.
  * Add common GPL and Apache license headers to copyright generator.
  * d/copyright: Regenerate.
```

### 6. Download the upstream tarball 
Use git-ubuntu to download the upstream tarball:
```
git ubuntu export-orig
```
This is a no-op if the tarball for the current upstream version is already present in the parent directory.

### 7. Create a .changes file.

Note the package version from the d/changelog. In the example above, it is 21.0.11+10. This corresponds to OpenJDK release 21.0.11+10. Here 11 is the minor version and 10 is the build number. Now go down the d/changelog and find the first entry that corresponds to a previous openjdk release. For example it could be: `21.0.10+9-2`). This is the `prev_package_release`.

Run `debuild` as follows:

- For the FIRST release built (the highest Ubuntu version), include the orig tarballs, or a PPA that does not already have them will reject the upload:
```
debuild -S -sa -d -uc -us -v<prev-package-release>
```
- For the remaining releases (the orig tarballs go in with the first upload):
```
debuild --no-lintian -S -d -uc -us -v<prev-package-release>
```

Plain `debuild -S` produces a "diff-only" upload (original source NOT included). Verify the orig tarballs are listed in the first .changes file:
```
grep -A10 '^Files:' ../openjdk-NN_<version>_source.changes
```
Alter the distribution value in the .changes file by removing the -security prefix. For example:
```
Distribution: resolute-security
```
should be changed to
```
Distribution: resolute
```

However, the changelog entry seen in the .changes file should not be changed. It should continue to have the -security prefix.


### 8. Push to the PPA

Skip this step. DO NOT sign the .changes file. DO NOT attempt to push to a ppa.
