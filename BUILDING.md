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
2. Packages the whole source tree as the release's source artifact —
   `target/cloudstore-<version>-src.tar.gz`, per
   `dev-support/assembly/source-release.xml` — since every ASF release
   needs at least one, and ATR checks for it;
3. Emits a CycloneDX SBOM next to the jar:
    - `target/cloudstore-<version>-cyclonedx.json`
    - `target/cloudstore-<version>-cyclonedx.xml`
4. Emits a SHA-256 digest next to the jar, the source tarball, and each
   SBOM file:
    - `target/cloudstore-<version>.jar.sha256`
    - `target/cloudstore-<version>-src.tar.gz.sha256`
    - `target/cloudstore-<version>-cyclonedx.json.sha256`
    - `target/cloudstore-<version>-cyclonedx.xml.sha256`

The SBOM is compile-scope only — `provided` dependencies are not in it because they are not shipped
in the jar. The source tarball ships plain `LICENSE`/`NOTICE` (not the
`-binary` variants, which list third-party binary dependencies that aren't
present in a source tree) and a `.rat-excludes` file at its root, so a RAT
scan of the extracted tarball — whether run by ATR or by hand — uses the
same exclusions as this project's own build.

The `.sha256` files are produced by `checksum-maven-plugin` in the `verify`
phase. They give downloaders a quick integrity check (`shasum -c
cloudstore-<version>.jar.sha256`) independent of the gpg signature.

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
and `.github/workflows/release.yml` must be registered under the *compose*
phase of cloudstore's ATR release policy. The full checklist, including the
repository variables to set (`ATR_HOST`, `ATR_PROJECT_KEY`, and friends),
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

The workflow, from a clean checkout of the ref you dispatched from:

1. builds with the `release` profile *twice*, independently, and fails the
   job unless the jar, the source tarball, and both SBOMs are all
   byte-identical between the two builds — this is what backs "anyone else
   building this gets the same binary", rather than just asserting it;
2. attests build provenance for all four artifacts via
   `actions/attest-build-provenance` (keyless, Sigstore/OIDC-backed —
   independent of, and in addition to, ATR's own signature);
3. signs all four artifacts with the committee's automated release
   signing key;
4. authenticates to ATR as a GitHub Actions Trusted Publisher (a GitHub
   OIDC token, exchanged for a 20-minute single-project SSH key) and
   uploads the signed artifacts, over rsync, into a new release candidate
   on ATR.

ATR then runs its own checks (signature, hash, archive structure, license,
RAT, SBOM conformance) against the uploaded candidate automatically; the job
summary points at where to review them.

ATR itself doesn't need a git tag — it tracks the candidate by project and
version — but tag the commit anyway so "Reproducing a release yourself"
below stays meaningful: `git tag v$ver <ref>; git push origin v$ver`.

### Phase 2: vote and publish (PMC, in ATR)

From here the process is entirely in ATR, not this repository:

1. Review the candidate and its check results on the ATR project page.
2. Start the vote from ATR — it emails dev@ with a link to the candidate on
   ATR (link only that page; see
   [Staging and voting](https://github.com/apache/tooling-trusted-releases/blob/main/atr/docs/staging-and-voting.md#what-to-link-in-a-vote-announcement)
   for why).
3. PMC members vote, independently rebuilding and comparing hashes as
   described below — this, not the automated signature, is what confirms
   the CI-built artifacts are genuine.
4. On a passing vote, finish the release from ATR. From Beta, ATR publishes
   directly to `dist/release`; during Alpha it publishes to `dist/atr` and a
   committer must move the files, as described in
   [Promoting to release](https://github.com/apache/tooling-trusted-releases/blob/main/atr/docs/promoting-to-release.md).

### Reproducing a release yourself

Anyone — a PMC member voting, or a third party after the fact — gets the
same binary from the same commit, provided they use the same JDK (Temurin
17, matching `release.yml`'s `Set up JDK` step; bytecode target is 1.8
either way):

```bash
git checkout v$ver
mvn -Prelease clean package -DskipTests
shasum -a 256 target/cloudstore-$ver.jar
# compare against the .sha256 published alongside the candidate/release on ATR
```

This works because `project.build.outputTimestamp` in `pom.xml` pins the
jar's internal timestamps to a fixed point (HEAD's commit time at release-cut,
set by `dev-support/bump-version.sh`) instead of "whenever the build ran" —
see the [Maven reproducible builds guide](https://maven.apache.org/guides/mini/guide-reproducible-builds.html).
Demonstrating exactly this was also the prerequisite for cloudstore's ATR
Trusted Publishing eligibility in the first place.

## Signing release artifacts manually

This is the fallback path: local smoke-testing of a release build, or
composing a candidate on ATR by hand (browser upload or personal-key rsync)
if Trusted Publishing isn't set up yet. `release.yml`'s automated compose
(above) does the equivalent signing itself, with the committee's automated
key, and does not need this.

Activate the `sign` profile alongside `release` to GPG-sign every attached
artifact with your own key. The plugin produces a detached `.asc` next to
each of:

- `target/cloudstore-<version>.jar`
- `target/cloudstore-<version>.pom`
- `target/cloudstore-<version>-src.tar.gz`
- `target/cloudstore-<version>-cyclonedx.json`
- `target/cloudstore-<version>-cyclonedx.xml`

Prerequisites:

1. An OpenPGP key in the Hadoop committer `KEYS` file.
2. `gpg-agent` running with the release key unlocked. A quick way to
   warm the agent before the build is:
   `echo test | gpg --clearsign -u <keyid> > /dev/null`.

Flags:

- `-Dgpg.keyName=<keyid>` — pick a specific key. If unset, gpg's
  default secret key is used.
- `-Dgpg.skip=true` — disable signing even when `-Psign` is active.

`-Prelease` on its own (without `sign`) still produces a valid jar and
SBOM — useful for local smoke tests on machines without the release
key.

Verify a downloaded release. The `.sha256` files contain the bare hex
digest (Apache convention) rather than the `shasum -c` wire format, so
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

*Important* Although the build tries to keep cloud credentials in `src/test/resources/auth-keys.xml` out of the source tarball,
along with all other build- and IDE-related artifacts, it is best to do a local release from a clean source directory not used
for test runs.
ATR is the preferred mechansim. 

## How to bypass buildnumber checks

```bash
mvn clean install -Prelease -DskipTests -Dbuildnumber.check=false -Dbuildnumber.update=false
```


