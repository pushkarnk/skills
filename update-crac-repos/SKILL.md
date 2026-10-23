---
name: update-crac-repos
description: Merge the crac repos with their upstreams
---

# Updating the CRaC repositories

## Background

Currently, OpenJDK CRaC on Ubuntu comprises of the following upstream repositories:
1. openjdk-17-crac at https://github.com/canonical/openjdk-17-crac
2. openjdk-21-crac at https://github.com/canonical/openjdk-21-crac
3. openjdk-25-crac at https://github.com/canonical/openjdk-25-crac
4. crac-criu at https://github.com/canonical/crac-criu

The openjdk-{17,21,25}-crac code is based on openjdk-{17,21,25}, and it needs to merged with every security update on the latter. The crac patches are picked up from https://github.com/openjdk/crac; the latter is a development repository and tracks the main-line openjdk/jdk repository. The crac patches for versions 17, 21 and 25 are selectively backported only if there is a failing JTREG test, or a user-reported issue. 

The crac-criu project is based on https://github.com/checkpoint-restore/criu and includes patches from https://github.com/CRaC/criu. We merge crac-criu with `checkpoint-criu/criu` on every new release of the latter. We selectively backport a new `CRaC/criu` patch only for a failing test or a user-reported failure.

## Instructions to update the CRaC repositories

These skill pertain to the merges only. The "selective backporting" is NOT a part of this skill.

### Merging crac-criu

1. Clone the crac-criu repository.
2. Add a new remote (`upstream`) pointing to https://github.com/checkpoint-restore/criu.
3. Fetch the latest tags from `upstream`.
4. Note the latest tag (`latest-tag`) from `upstream`.
5. Checkout a new `crac-criu-<latest-tag>` branch off the `crac` branch.
6. Run a git merge: `git merge <latest-tag>`
7. Fix merge conflicts if any.
8. Run a build:
   - Install the following build-dependencies: libbsd-dev, libcap-dev, libgnutls28-dev, libnet1-dev, libnftables-dev, libnl-3-dev, libprotobuf-c-dev, libprotobuf-dev, pkgconf, protobuf-c-compiler, protobuf-compiler, uuid-dev
   - Run `make`
   - A successful build should create a binary: src/criu/criu

### Merging openjdk-{17,21,25}-crac

Follow these steps for each of the three packages -17, -21 and -25:

1. Clone the openjdk-NN-crac repository
2. Add https://github.com/openjdk/jdkNNu as a remote (e.g: https://github.com/openjdk/jdk17u)
3. Get the latest tag from `openjdk/jdkNNu` with "-ga" in it , and now get the tag previous to it.
   For example: the latest ga tag is jdk-17.0.20-ga, and the correct tag is jdk-17.0.20+8 which has the _same_ commit hash (could be before or after the ga tag). Note this tag as the `merge-tag`.
4. Create a new branch named `crac-<merge-tag>`.
5. Run a git merge: `git merge <merge-tag>`
6. If there are merge conflicts, resolve them and complete the merge.
7. Run a build:
   - Install `openjdk-NN-jdk` using apt
   - Run `bash configure --disable-warnings-as-errors`. Until all the build depdendencies are installed, this command will fail.
     But it will also provide the `apt` commands to install those dependencies. Keep installing the build dependencies and repeating this command until it passes.
   - Run `make images`
   - If the build fails, abort and report failure.
   - After this command works properly, you can also configure `jtreg` directly with `bash configure --with-jtreg=/usr/share/jtreg` or some other installation directory.
8. Run tests:
   - Install `jtregN` using `apt` if needed. Package -17 uses `jtreg7` and packages -21 and -25 use `jtreg8`
   - Install `crac-criu` using `apt`. Copy `/usr/sbin/crac-criu` into `build/linux-x86_64-server-release/jdk/lib`
   - Run these two commands:
     - `sudo chown root:root build/linux-x86_64-server-release/jdk/lib/criu`
     - `sudo chmod u+s build/linux-x86_64-server-release/jdk/lib/criu`
   - Run the crac tests: ` sudo jtreg -v1 -agentvm -testjdk:build/linux-x86_64-server-release/jdk/ test/jdk/jdk/crac`
     Note: `sudo` is necessary on a corporate laptop which has authd-managed Ubuntu.
     Note: `-agentvm` is required to replicate the test environment that CI currently uses.
   - Report the failing tests.
   - Next, replace `build/linux-x86_64-server-release/jdk/lib/criu` by `crac-criu/criu/criu` for each of
     openjdk-{17,21,25}-crac and re-run the tests. Report any new failures.

> Note: The CI for testing openjdk-{17,21,25}-crac currently uses the `make test TEST='jdk/crac'` fixture. This may present alternative test results that what can be replicated locally. Additionally, this test uses an alternative location for `criu` than previously tested against so it may feature failures locally due to a missing `criu` executable.
