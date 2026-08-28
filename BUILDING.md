<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

# Building


# Hadoop versions

There's a dependency on hadoop 3.4.2 to be compatible with its avro version and to
have non-reflective access to the bulk delete API.
You should still code targeting hadoop 3.4.0 as the minimum version *outside these specific
cli commands*

## Compiling

Builds with Apache Maven

To build a production release
1. Compile on a JDK17 JVM; the output is still generated for Java 8 JVMs.
2. Compile against a shipping hadoop version (see the profiles).


```bash
mvn spotless:apply              # auto-format Java sources to palantir-java-format
mvn clean install               # compile + unit tests + jar
mvn clean verify                # adds: ITest*, apache-rat:check, spotless:check
mvn org.apache.rat:apache-rat-plugin:check    # license-header audit only
mvn site                        # render src/site → target/site (fluido skin)
```

The `verify` lifecycle is the one CI runs. To run cloud-backed contract tests
opt in via profile + credentials in `src/test/resources/auth-keys.xml`:

## Testing

There are unit tests for store operations against the local fs and a mini hdfs cluster.

There are integration tests which run with the `mvn verify` command, currently against S3 stores only.
The tests are minimal, any basic S3 implementation will suffice.

To bind to a store, follow the [testing s3a](https://hadoop.apache.org/docs/stable/hadoop-aws/tools/hadoop-aws/testing.html#File_auth-keys.xml) docs.

## Updating cloudstore release versions

For a long time the artifact version was fixed at 1.0 so that curl and other
tools could fetch a stable URL. That was convenient but produced a different
problem: support tickets kept attaching the same `cloudstore-<old>.jar`
long after new releases had shipped, with no version change to make the staleness
visible. Release number increments are now required for anything other than
a rapid-iteration multiple-releases-in-a-day workflow.

The project follows the conventional Maven SNAPSHOT lifecycle:

| Phase                     | pom `<version>`    | docs (`cloudstore-X.Y.jar`) |
|---------------------------|--------------------|-----------------------------|
| Day-to-day development    | `X.Y-SNAPSHOT`     | last released `X.Y`         |
| Cutting release `X.Y`     | `X.Y`              | `X.Y`                       |
| Immediately after release | `X.(Y+1)-SNAPSHOT` | `X.Y`                       |

`-SNAPSHOT` appears in the pom only; the published artifact and every
documented `cloudstore-X.Y.jar` reference is always a bare release form.

Two scripts in `dev-support/` keep these in sync:


**`bump-version.sh <new-version>`**

Bumps `pom.xml`'s `<version>` only.
Accepts either `X.Y-SNAPSHOT` (for development bumps) or `X.Y`
(when cutting a release).
Also rewrites `BUILDING.md`'s `set -gx ver <v>` line so the release command block
below stays in step (the line always reflects the bare release
version — a trailing `-SNAPSHOT` is stripped before substituting).
Does *not* touch `README.md`, `AGENTS.md`, or `src/site/markdown/*`.

**`update-site-docs.sh <new-version>`**
Updates site documentation for releases.
Rewrites every `cloudstore-<old>.jar` /
  `cloudstore-<old>-cyclonedx` reference in `README.md`, `AGENTS.md`,
  `BUILDING.md`, and `src/site/markdown/*.md`, and bumps the
  `<cloudstore.docs.version>` property in `pom.xml`.
Rejects attempts to switch to a `-SNAPSHOT`; SNAPSHOT artifacts are not released and so not documented.

The `verify` phase enforces this: `dev-support/check-doc-versions.sh`
runs as part of `mvn verify` and fails the build if any markdown doc
references a `cloudstore-X.Y.jar` that disagrees with
`${cloudstore.docs.version}` (independent of `${project.version}`, so
SNAPSHOT pom states do not trip the gate).

### Worked example: cutting release 1.4 then returning to dev

```bash
# 1. Cut the release: pom 1.4-SNAPSHOT -> 1.4, site docs 1.3 -> 1.4.
dev-support/bump-version.sh 1.4
dev-support/update-site-docs.sh 1.4
# (build, tag, publish — see "Releasing" below.)

# 2. Back to development: pom 1.4 -> 1.5-SNAPSHOT. Docs stay at 1.4.
dev-support/bump-version.sh 1.5-SNAPSHOT
```


## Releasing

Cutting a release targets [Apache Trusted Releases](https://github.com/apache/tooling-trusted-releases)
(ATR), the ASF's platform for composing, voting on, and publishing official
releases. It is two phases: an **automated** build+compose phase that runs
in CI and uploads a release candidate to ATR, and a **manual** vote+publish
phase performed by the PMC through ATR itself. No private key lives outside
ATR's own infrastructure: the compose phase signs with a dedicated
"Automated Release Signing" key that ASF Infra provisions specifically for
unattended CI use (see the setup checklist in `.github/workflows/release.yml`'s
header comment), and voting/publishing happen as ASF release policy
requires — a PMC vote is not something CI can substitute for.

Release builds activate the `release` profile, which:

1. Enforces a clean git tree via `buildnumber-maven-plugin`;
2. Emits a CycloneDX SBOM next to the jar:
    - `target/cloudstore-<version>-cyclonedx.json`
    - `target/cloudstore-<version>-cyclonedx.xml`
3. Emits a SHA-256 digest next to the jar and each SBOM file:
    - `target/cloudstore-<version>.jar.sha256`
    - `target/cloudstore-<version>-cyclonedx.json.sha256`
    - `target/cloudstore-<version>-cyclonedx.xml.sha256`

The SBOM is compile-scope only — `provided` dependencies are not in it because they are not shipped
in the jar.

The `.sha256` files are produced by `checksum-maven-plugin` in the `verify`
phase. They give downloaders a quick integrity check (`shasum -c
cloudstore-<version>.jar.sha256`) independent of the gpg signature.

### The source artifact: two independent paths, by design

Every ASF release needs at least one source artifact, and ATR checks for
it. Cloudstore has two ways to build it's source file `cloudstore-<version>-src.tar.gz`


- **`release.yml`'s ATR upload path** uses `git archive` to snapshot the
  source at `HEAD`.
- **`mvn -Passembly`, alongside `-Prelease`**
  Packages the working directory via `maven-assembly-plugin`
  (`dev-support/assembly/source-release.xml`), a filesystem excludes list
  rather than `git ls-files`, so it works from a shallow clone, a fork, or
  a directory that isn't a git checkout at all. `mvn -Prelease clean
  verify` on its own does *not* build this artifact.

It also generates a build version, with the buildnumber plugin.
On release builds, this will fail the build if there are uncommitted changes.

** You must commit all changes before starting a release build.**

The build will fail if there are uncommitted changes.

```
[INFO] --- buildnumber:3.3.0:create (default) @ cloudstore ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.344 s
[INFO] Finished at: 2026-06-26T17:23:21+01:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.codehaus.mojo:buildnumber-maven-plugin:3.3.0:create (default) on project cloudstore:
        Cannot create the build number because you have local modifications : 
```

### One-time setup

Before this workflow can be used at all, cloudstore's committee needs to be
onboarded to ATR Trusted Publishing: ASF Security must confirm the build is
reproducible, ASF Infra must cut the committee's "Automated Release Signing"
key and store its private half as the `ATR_SIGNING_KEY` repository secret,
a repository variable `ATR_PROJECT_KEY` must be set to cloudstore's project
key in ATR, and `.github/workflows/release.yml` must be registered under
the *compose* phase of cloudstore's ATR release policy. The full checklist
lives in that workflow file's header comment rather than being duplicated
here — check there for the current state of that setup.

### Phase 1: automated build + compose (CI)

After `dev-support/bump-version.sh <version>` and
`dev-support/update-site-docs.sh <version>` have been run and committed
(see the worked example above), dispatch `.github/workflows/release.yml`:

```bash
set -gx ver 1.6                           # version being cut
gh workflow run release.yml -f version=$ver
gh run watch (gh run list --workflow=release.yml -L1 --json databaseId -q '.[0].databaseId')
```

The workflow is modeled on
[log4j's working ATR deploy workflow](https://github.com/apache/logging-log4j2/blob/release-atr/2.26.1/.github/workflows/deploy-atr.yaml),
reusing the same building blocks.
It

1. Builds the jar and SBOMs with the `release` profile *twice*,
   independently, rebuilding the source tarball with `git archive` each
   time too, and fails the job unless all four are byte-identical between
   the two builds.
2. Attests build provenance for all four artifacts via
   `actions/attest-build-provenance`.
3. Imports the committee's automated release signing key with
   [`crazy-max/ghaction-import-gpg`](https://github.com/crazy-max/ghaction-import-gpg)
   and signs all staged artifacts;
4. Uploads them to ATR with
   [`apache/tooling-actions/upload-to-atr`](https://github.com/apache/tooling-actions).

ATR then runs its own checks (signature, hash, archive structure, license,
RAT, SBOM conformance) against the uploaded candidate automatically; the job
summary points at where to review them.

ATR tracks the candidate by project and
version — but do tag the commit anyway so "Reproducing a release yourself"
below stays meaningful: `git tag v$ver <ref>; git push origin v$ver`.

### Phase 2: vote and publish (PMC, in ATR)

From here the process is entirely in ATR, not this repository:

1. Review the candidate and its check results on the ATR project page.
2. Start the vote from ATR — it emails dev@ with a link to the candidate on
   ATR (link only that page; see
   [Staging and voting](https://github.com/apache/tooling-trusted-releases/blob/main/atr/docs/staging-and-voting.md#what-to-link-in-a-vote-announcement)
   for why).
3. Developers (nonbinding) and PMC members vote (binding).
4. On a passing vote, finish the release from ATR. From Beta, ATR publishes
   directly to `dist/release`; during Alpha it publishes to `dist/atr` and a
   committer must move the files, as described in
   [Promoting to release](https://github.com/apache/tooling-trusted-releases/blob/main/atr/docs/promoting-to-release.md).


## Building and signing release artifacts locally

This is needed for local builds.

Activate the `sign` profile alongside `release` to GPG-sign every
Maven-attached artifact with your own key. The plugin produces a detached
`.asc` next to each of:

- `target/cloudstore-<version>.jar`
- `target/cloudstore-<version>.pom`
- `target/cloudstore-<version>-cyclonedx.json`
- `target/cloudstore-<version>-cyclonedx.xml`

Add `-Passembly` too (`mvn -Prelease,assembly,sign ...`) and the source
tarball is Maven-attached like the rest, so `-Psign` signs it the same way.

Prerequisites:

1. An OpenPGP key in the Hadoop committer [KEYS](https://downloads.apache.org/hadoop/common/KEYS) file.
2. `gpg-agent` running with the release key unlocked.

Flags:

- `-Dgpg.keyName=<keyid>` — pick a specific key. If unset, gpg's
  default secret key is used.
- `-Dgpg.skip=true` — disable signing even when `-Psign` is active.

`-Prelease` on its own (without `sign`) still produces a valid jar and
SBOM — useful for local smoke tests on machines without the release
key.

Verify a downloaded release. The `.sha256` files contain the bare hex
digest rather than the `shasum -c` wire format, so
compare directly:

```bash
# integrity: hash matches what the release published.
[ "$(shasum -a 256 cloudstore-$ver.jar | awk '{print $1}')" = \
  "$(cat cloudstore-$ver.jar.sha256)" ] && echo "sha256 OK"
# authenticity: the matching signature is from the expected key.
gpg --verify cloudstore-$ver.jar.asc cloudstore-$ver.jar
```

To upload artifacts signed this way to ATR by hand, see the "How files reach
ATR" section of
[Staging and voting](https://github.com/apache/tooling-trusted-releases/blob/main/atr/docs/staging-and-voting.md#how-files-reach-atr).


## How to bypass buildnumber checks

```bash
mvn clean install -Prelease -DskipTests -Dbuildnumber.check=false -Dbuildnumber.update=false
```


