---
name: gather-jtreg-results-private-ppa
description: Gather JTREG results from a private Ubuntu PPA and tabulate them
---

# Skill to gather and tabulate JTREG test results
OpenJDK builds are run in a private PPA on Launchpad. This skill is used to collect all jtreg results for a given JDK version and tabulate them. The skills also needs to be supplied the name of the PPA and the KEYID (fingerprint of the PPA's signing key).

## Skill arguments

$1 - OpenJDK version (openjdk-11, openjdk-21 etc.)
$2 - Quarterly update tag (25.0.4+7)
$3 - Launchpad username
$4 - Launchpad token as per the https://launchpad.net/~pushkarnk/+archivesubscriptions page.
$5 - PPA name (For ppa:stonking-ppa/ppa, pass stonking-ppa/ppa)

## Legends

### Support matrix
Please note the openjdk package versions and the supported Ubuntu releases:

| OpenJDK version | List of supported Ubuntu versions that need the package update | Do JTREG analysis? |
| --- | --- | --- |
| openjdk-26 | 26.04 | Y | 
| openjdk-25 | 26.04, 24.04, 22.04 | Y |
| openjdk-21 | 26.04, 24.04, 22.04, 20.04 | Y |
| openjdk-17 | 26.04, 24.04, 22.04, 20.04, 18.04 | Y |
| openjdk-8  | 26.04, 24.04, 22.04, 20.04, 18.04, 16.04 | N |

### Arch list
List of architectures:
amd64
arm64
armhf
i386
riscv64
ppc64el
s390x

## Download debs containing the JTREG results

Do this for every supported Ubuntu version for $1
1. Create a directory named openjdk-NN-<ubuntu-version> as per $1 and `cd` into it.
2. For every architecture in the "Arch list" legend download the jdk deb using the following command template:

   wget https://$3:$4@private-ppa.launchpad.net/$5/ubuntu/pool/main/o/openjdk-NN/openjdk-NN-jdk_$2-<package-version>~<ubuntu-version>[.k]_{arch}.deb

   Package version could be: 1, 2, 0ubuntu1, 1ubuntu2 etc.
   The value of k could be 1, 2, 3 etc.

   Note: the version infix may contain extra segments, e.g.
   `openjdk-8-jdk_8u492-ga~us2-0ubuntu1~22.04.1_amd64.deb`.

### If the debs are already downloaded

If the user has already downloaded the debs (e.g. into the current working
directory), skip the download step entirely. Work directly with the .deb
files present: unpack each into its own directory (e.g. `unpacked/<deb-name>/`)
and proceed to the parsing step below.

## Parse JTREG results for each (ubuntu-release, architecture) pair

1. Unpack each .deb using `dpkg-deb -x <deb> <dir>`
2. Read the file named `usr/share/doc/openjdk-NN-jre-headless/test-{arch}/jtreg-summary-hotspot.log` in the unpacked directory.
   Note: this file is gzip-compressed in some packages (`jtreg-summary-hotspot.log.gz`); use `zcat` to read it.
   Note: this file may not exist in some debs. Make a note and continue with the next .deb.
3. The file has up to 4 sections: hotspot, jaxp, langtools, jdk (openjdk-8 has no jaxp section).
   Each section is retried twice, so it contains 3 rounds of failure lists, each round ending
   with a `Test results:` line. The definitive list of failing tests for a section is the
   block of `FAILED:`/`Error:` lines immediately before the LAST `Test results:` line of
   that section.

## Tabulate results

Create a table of results in this format:
| Ubuntu-version | Arch | Failing tests |