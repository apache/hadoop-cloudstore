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

Cutting a release is two phases: an **automated** build+stage phase that runs
in CI, and a **manual** sign/vote/publish phase performed by the release
manager (RM). No private key lives in CI — GPG signing with the RM's
personal ASF key remains a human step, as ASF release policy requires.
What CI automates is everything before that: building, proving the build is
reproducible, attesting its provenance, and staging the result as a draft
GitHub Release.

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

### Phase 1: automated build + staging (CI)

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
   job unless the jar and both SBOMs are byte-identical between the two
   builds — this is what backs "anyone else building this gets the same
   binary", rather than just asserting it;
2. attests build provenance for the jar and SBOMs via
   `actions/attest-build-provenance` (keyless, Sigstore/OIDC-backed — no
   secret key stored in CI);
3. stages everything as a **draft** GitHub Release tagged `v<version>`
   (superseding the old `tag-release-<timestamp>` scheme — one tag per
   release version now, since multiple runs for the same version just
   update the same draft).

The job summary lists the Phase 2 steps below as a checklist.

### Phase 2: manual sign, vote, publish (RM)

```bash
test -n "$ver"; or begin; echo "ERROR: \$ver is unset/empty -- run: set -gx ver <version>"; exit 1; end

# Rebuild locally and sign -- reproducibility means this jar is byte-identical
# to the one CI already staged, so signing it is equivalent to signing CI's.
mvn clean install -Prelease,sign -DskipTests

# Independent confirmation before trusting it: compare against the draft's
# published digest.
gh release download v$ver -D /tmp/release-$ver --pattern '*.sha256'
[ "$(shasum -a 256 target/cloudstore-$ver.jar | awk '{print $1}')" = \
  "$(cat /tmp/release-$ver/cloudstore-$ver.jar.sha256)" ] && echo "reproducible: matches CI's build"

# Upload just the signatures -- the jar/SBOMs/checksums are already staged.
gh release upload v$ver \
    target/cloudstore-$ver.jar.asc \
    target/cloudstore-$ver-cyclonedx.json.asc \
    target/cloudstore-$ver-cyclonedx.xml.asc

echo "go to the web ui to review, start the dev@ vote referencing the draft"
echo "and its provenance attestation (gh attestation verify), and on a"
echo "passing vote: gh release edit v$ver --draft=false"
```

### Reproducing a release yourself

Anyone — CI, the RM, or a third party — gets the same binary from the same
commit, provided they use the same JDK (Temurin 17, matching
`release.yml`'s `Set up JDK` step; bytecode target is 1.8 either way):

```bash
git checkout v$ver
mvn -Prelease clean package -DskipTests
[ "$(shasum -a 256 target/cloudstore-$ver.jar | awk '{print $1}')" = \
  "$(curl -sL https://github.com/apache/hadoop-cloudstore/releases/download/v$ver/cloudstore-$ver.jar.sha256)" ] \
  && echo "reproducible"
```

This works because `project.build.outputTimestamp` in `pom.xml` pins the
jar's internal timestamps to a fixed point (HEAD's commit time at release-cut,
set by `dev-support/bump-version.sh`) instead of "whenever the build ran" —
see the [Maven reproducible builds guide](https://maven.apache.org/guides/mini/guide-reproducible-builds.html).

## Signing release artifacts

Activate the `sign` profile alongside `release` (as in the command
above) to GPG-sign every attached artifact. The plugin produces a
detached `.asc` next to each of:

- `target/cloudstore-<version>.jar`
- `target/cloudstore-<version>.pom`
- `target/cloudstore-<version>-cyclonedx.json`
- `target/cloudstore-<version>-cyclonedx.xml`

Prerequisites:

1. An OpenPGP key in the Hadoop committer KEYS file.
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

Phase 1 above already stages the jar, SBOMs, and `.sha256` digests on the
draft release; Phase 2's `gh release upload v$ver *.asc` is what attaches
the signatures produced here.

## How to bypass buildnumber checks

```bash
mvn clean install -Prelease -DskipTests -Dbuildnumber.check=false -Dbuildnumber.update=false
```


