# ZhiJun IO Workflows

Shared infrastructure for ZhiJun IO Maven projects: reusable GitHub Actions workflows and release script.

## Repository Structure

```
workflows/
├── .github/
│   ├── workflows/
│   │   ├── maven-ci.yml
│   │   ├── maven-snapshot.yml
│   │   └── maven-release.yml
│   ├── actions/
│   │   ├── run-maven/                  # mvn / mvnw + profiles, GPG
│   │   └── set-project-version/        # versions:set or <revision> (auto)
├── examples/rose/                      # Rose integration samples
└── zhijun-io-release.py
```

## Quick Start

Reference workflows from this repo:

```yaml
uses: zhijun-io/workflows/.github/workflows/maven-ci.yml@main
```

During development, pin to a branch:

```yaml
uses: zhijun-io/workflows/.github/workflows/maven-ci.yml@feature/my-branch
```

### 1. CI (`.github/workflows/maven-ci.yml`)

```yaml
name: Maven CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: zhijun-io/workflows/.github/workflows/maven-ci.yml@main
```

### 2. Snapshot publish

```yaml
name: Maven Publish Snapshot
on:
  workflow_dispatch:

jobs:
  publish:
    uses: zhijun-io/workflows/.github/workflows/maven-snapshot.yml@main
    secrets:
      MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
      MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
      MAVEN_GPG_PRIVATE_KEY: ${{ secrets.MAVEN_GPG_PRIVATE_KEY }}
      MAVEN_GPG_PASSPHRASE: ${{ secrets.MAVEN_GPG_PASSPHRASE }}
```

### 3. Maven Central release

```yaml
name: Maven Central Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Release version'
        required: true
        type: string

jobs:
  release:
    permissions:
      contents: write
    uses: zhijun-io/workflows/.github/workflows/maven-release.yml@main
    with:
      version: ${{ inputs.version }}
    secrets:
      MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
      MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
      MAVEN_GPG_PRIVATE_KEY: ${{ secrets.MAVEN_GPG_PRIVATE_KEY }}
      MAVEN_GPG_PASSPHRASE: ${{ secrets.MAVEN_GPG_PASSPHRASE }}
```

### Release script

```bash
python3 zhijun-io-release.py rose 0.1.0 --dry-run
python3 zhijun-io-release.py rose 0.1.0
```

## Reusable Workflows

JDK setup uses [`actions/setup-java@v4`](https://github.com/actions/setup-java) directly in each workflow.

### Common inputs

| Input | Default | Used in |
|-------|---------|---------|
| `working-directory` | `.` | CI, snapshot, release |
| `java-version` | `21` | CI, snapshot, release |
| `java-distribution` | `temurin` | CI, snapshot, release |
| `use-maven-wrapper` | `true` | CI, snapshot, release |
| `skip-tests` | `false` | CI |
| `maven-profiles` | *(empty)* | CI, snapshot, release (verify and deploy) |
| `maven-extra-args` | `-B -ntp` | CI, snapshot, release |
| `maven-server-id` | `central` | snapshot, release |

### CI (`maven-ci.yml`)

```yaml
jobs:
  build:
    uses: zhijun-io/workflows/.github/workflows/maven-ci.yml@main
    with:
      java-version: '21'
      maven-goals: clean verify
      # maven-profiles: your-profile   # only if defined in your POM
      upload-test-results: true
      upload-coverage: false
      run-sonar: false
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # when run-sonar is true
```

| Input | Default | Description |
|-------|---------|-------------|
| `maven-goals` | `clean verify` | Maven goals to run |
| `upload-test-results` | `true` | Upload Surefire/Failsafe reports |
| `upload-coverage` | `false` | Upload JaCoCo site/exec artifacts |
| `run-sonar` | `false` | Sonar scan in a separate job via SonarScanner for Maven (JDK 21 by default) |
| `sonar-java-version` | `21` | Java version for the Sonar job only |
| `sonar-host-url` | `https://sonarcloud.io` | Sonar server URL |
| `sonar-organization` | *(empty)* | SonarCloud organization; default = GitHub owner on SonarCloud; omit on self-hosted |
| `sonar-project-key` | *(empty)* | Sonar project key; default = GitHub repository name |
| `sonar-coverage-report-paths` | *(empty)* | JaCoCo XML paths for Sonar |
| `sonar-maven-plugin-version` | `5.7.0.6970` | SonarScanner for Maven plugin version |

Sonar runs in job **`sonar`** after **`build`**, using `sonar-java-version` (default `21`) and **`org.sonarsource.scanner.maven:sonar-maven-plugin`**, so projects can build on Java 8 and scan on Java 21. The build job uploads `target/classes`, JaCoCo outputs, and test reports for analysis.

Override Sonar connection via workflow inputs (or `pom.xml` properties):

```yaml
with:
  run-sonar: true
  sonar-host-url: https://sonarcloud.io
  sonar-organization: my-org        # omit for GitHub owner
  sonar-project-key: my-project     # omit for repository name
```

When `run-sonar: true`, the **caller workflow** must grant `pull-requests: read` (Sonar job uses it for PR decoration):

```yaml
permissions:
  contents: read
  pull-requests: read

jobs:
  build:
    uses: zhijun-io/workflows/.github/workflows/maven-ci.yml@main
    with:
      java-version: '8'
      run-sonar: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### Publish Snapshot (`maven-snapshot.yml`)

```yaml
jobs:
  publish:
    uses: zhijun-io/workflows/.github/workflows/maven-snapshot.yml@main
    with:
      maven-profiles: release   # if Central/GPG plugins live in release profile
    secrets:
      MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
      MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
      MAVEN_GPG_PRIVATE_KEY: ${{ secrets.MAVEN_GPG_PRIVATE_KEY }}
      MAVEN_GPG_PASSPHRASE: ${{ secrets.MAVEN_GPG_PASSPHRASE }}
```

GPG secrets are optional but required for signed Central deployments.

### Maven Central Release (`maven-release.yml`)

Set version → **commit release** → deploy → **push release commit** → GitHub Release (tag) → bump next SNAPSHOT → push bump.

```yaml
jobs:
  release:
    permissions:
      contents: write   # required: push commits, create GitHub Release
    uses: zhijun-io/workflows/.github/workflows/maven-release.yml@main
    with:
      version: ${{ inputs.version }}
      maven-profiles: release   # if Central/GPG plugins live in release profile
    secrets:
      MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
      MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
      MAVEN_GPG_PRIVATE_KEY: ${{ secrets.MAVEN_GPG_PRIVATE_KEY }}
      MAVEN_GPG_PASSPHRASE: ${{ secrets.MAVEN_GPG_PASSPHRASE }}
```

| Input | Default | Description |
|-------|---------|-------------|
| `version` | *(required)* | Release version (not SNAPSHOT) |
| `version-property` | `auto` | `auto`, `project-version`, or `revision` |
| `pom-file` | `pom.xml` | POM path relative to working-directory |
| `create-tag` | `true` | Create GitHub Release and tag on the **release commit** |
| `tag-prefix` | `v` | Tag prefix (e.g. `v0.1.0`) |
| `bump-snapshot` | `true` | Commit next SNAPSHOT after release (independent of `create-tag`) |
| `next-snapshot-version` | *(empty)* | Override patch+1 SNAPSHOT bump |

**Default branch only:** always checks out the repository default branch (`gh repo view … defaultBranchRef`), regardless of which branch triggered `workflow_dispatch`. Only run release from default branch when it matches what you intend to ship.

**Git flow:** the release commit contains the exact version published to Central; the tag (`create-tag: true`) points to that commit via `target_commitish`. With `bump-snapshot: false`, default branch stays on the release version until you bump manually.

**Caller permissions:** the calling job must grant `contents: write` (see example above). Repositories with read-only default `GITHUB_TOKEN` will fail on push or GitHub Release.

### Release partial failures

| Failed after… | Central | Git default branch | Tag / Release | Remediation |
|---------------|---------|-------------------|---------------|-------------|
| deploy | maybe published | release commit not pushed | — | Check Central; do not re-deploy same version — fix and release next patch |
| push release commit | published | unchanged | — | Manually push the release commit or revert Central if unpublished |
| GitHub Release | published | release commit pushed | missing | Create tag/release manually at the release commit SHA |
| SNAPSHOT bump / push | published | stuck on release version | tag OK | Manually bump POM to next `-SNAPSHOT` and push |

## Versioning

| Mode | When to use | How it works |
|------|-------------|--------------|
| **`auto`** (default) | Most projects | `<revision>` in `<properties>` → sed; otherwise `versions:set` |
| **`project-version`** | Standard `<version>` in POM | `versions-maven-plugin:set` |
| **`revision`** | Rose-style `${revision}` | sed on `<revision>` property |

**No `${revision}`?** Default `auto` works with `<version>1.0.0-SNAPSHOT</version>` — no flatten plugin needed.

**With `${revision}`:**

```xml
<properties>
  <revision>0.1.0-SNAPSHOT</revision>
</properties>
<version>${revision}</version>
```

Optionally add `flatten-maven-plugin` for CI-friendly installed POMs (Rose pattern).

## Composite Actions

| Action | Purpose |
|--------|---------|
| `run-maven` | Run `mvn` / `mvnw` with profiles, skipTests, extra-args |
| `set-project-version` | `versions:set` or `<revision>` (auto-detect); optional `expected-version` verify |

Release SNAPSHOT/semver input checks run inline in `maven-release.yml`.

## Secrets

Configure at **organization** or **repository** level: Settings → Secrets and variables → Actions.

| Secret | Required for | Description |
|--------|--------------|-------------|
| `MAVEN_USERNAME` | Snapshot, Release | Sonatype Central Portal username — [central.sonatype.com](https://central.sonatype.com/) |
| `MAVEN_PASSWORD` | Snapshot, Release | Sonatype Central Portal token (not account password) |
| `MAVEN_GPG_PRIVATE_KEY` | Signed publish / Release | ASCII-armored GPG private key |
| `MAVEN_GPG_PASSPHRASE` | Signed publish / Release | GPG key passphrase |
| `SONAR_TOKEN` | CI Sonar (`run-sonar: true`) | Sonar server token (SonarCloud or self-hosted) |

### Setup

1. **Sonatype Central** — register at https://central.sonatype.com/, verify email, register your `groupId` namespace.
2. **Generate GPG key** (if not already):

```bash
gpg --full-generate-key
gpg --list-secret-keys --keyid-format LONG
gpg --armor --export-secret-keys KEY_ID > private-key.asc
# Entire private-key.asc → MAVEN_GPG_PRIVATE_KEY secret
```

3. **Add secrets in GitHub** — Settings → Secrets and variables → Actions → New repository secret (or org-level for all repos).

### Verify

| Workflow | How to test |
|----------|-------------|
| Snapshot | Push to default branch, or Actions → Maven Publish Snapshot → Run workflow |
| Release | Actions → Maven Central Release → Run workflow → enter version (e.g. `0.1.0`) |
| Sonar | CI with `run-sonar: true` and `SONAR_TOKEN` set |

Check publish status in GitHub Actions logs and https://central.sonatype.com/.

### Troubleshooting

| Error | Likely cause | Fix |
|-------|--------------|-----|
| `401 Unauthorized` | Wrong Maven credentials | Re-login to Central Portal; update `MAVEN_USERNAME` / `MAVEN_PASSWORD` |
| `gpg: signing failed: No secret key` | Invalid `MAVEN_GPG_PRIVATE_KEY` | Use armored format with `BEGIN/END PGP PRIVATE KEY`; copy full key including newlines |
| `Inappropriate ioctl for device` / loopback errors | CI pinentry / agent mismatch | `setup-java` `gpg-passphrase` must be env var **name** (`MAVEN_GPG_PASSPHRASE`), not the secret value; set `env.MAVEN_GPG_PASSPHRASE`; `run-maven` uses loopback + `-Dgpg.use.agent=false` |
| `403 Forbidden` | No publish permission | Verify Sonatype account; confirm `groupId` is registered to your namespace |

### Security

- Never commit secrets to the repository or print them in logs.
- Prefer org-level secrets shared across ZhiJun IO repos; rotate GPG keys and tokens periodically.
- Use Portal **tokens** for `MAVEN_PASSWORD`, not your login password.

### Links

- [GitHub encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Sonatype Central publish guide](https://central.sonatype.org/publish/publish-guide/)
- [Generating a GPG key (GitHub docs)](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key)

## POM Requirements

1. Maven wrapper (`./mvnw`) or `use-maven-wrapper: false`
2. `release` profile with Central publishing plugin, GPG, sources, javadoc — pass `maven-profiles: release` in snapshot/release workflows when plugins are profile-scoped
3. `maven-enforcer-plugin` with `requireReleaseDeps` (blocks SNAPSHOT dependencies during `verify`)

Example enforcer:

```xml
<plugin>
  <artifactId>maven-enforcer-plugin</artifactId>
  <version>3.5.0</version>
  <executions>
    <execution>
      <id>enforce-no-snapshot-dependencies</id>
      <goals><goal>enforce</goal></goals>
      <phase>validate</phase>
      <configuration>
        <rules>
          <requireReleaseDeps>
            <message>No SNAPSHOT dependencies allowed</message>
          </requireReleaseDeps>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Example `release` profile:

```xml
<profile>
  <id>release</id>
  <build>
    <plugins>
      <plugin>
        <groupId>org.sonatype.central</groupId>
        <artifactId>central-publishing-maven-plugin</artifactId>
        <version>0.9.0</version>
        <extensions>true</extensions>
        <configuration>
          <publishingServerId>central</publishingServerId>
          <autoPublish>true</autoPublish>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-gpg-plugin</artifactId>
        <version>3.2.7</version>
        <executions>
          <execution>
            <id>sign-artifacts</id>
            <phase>verify</phase>
            <goals><goal>sign</goal></goals>
            <configuration>
              <gpgArguments>
                <arg>--pinentry-mode</arg>
                <arg>loopback</arg>
              </gpgArguments>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</profile>
```
