# ZhiJun IO Workflows

Shared infrastructure for ZhiJun IO projects including reusable GitHub Actions workflows, release scripts, and project configuration templates.

## Repository Structure

```
workflows/
├── .github/
│   ├── workflows/
│   │   ├── maven-ci.yml
│   │   ├── maven-snapshot.yml
│   │   └── maven-release.yml
│   ├── actions/
│   │   ├── run-maven/                        # Composite: mvn / mvnw
│   │   ├── set-project-version/              # versions:set or <revision>
│   │   └── verify-no-snapshot-versions/
│   ├── community-projects.yml              # Project registry
│   └── project.yml.template                # Template for project config
├── examples/                               # Example configurations
├── zhijun-io-release.py          # Release script
├── RELEASE.md                              # Detailed release documentation
└── README.md                               # This file
```

## Quick Start

### For New Projects

1. **Add CI workflow** (`.github/workflows/maven-ci.yml`):
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

2. **Add snapshot publishing** (`.github/workflows/maven-snapshot.yml`):
```yaml
name: Maven Publish Snapshot
on:
  push:
    branches: [main]

jobs:
  publish:
    uses: zhijun-io/workflows/.github/workflows/maven-snapshot.yml@main
    secrets:
      MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
      MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
```

3. **Add release workflow** (`.github/workflows/maven-release.yml`):
```yaml
name: Maven Central Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Release version'
        required: true

jobs:
  release:
    uses: zhijun-io/workflows/.github/workflows/maven-release.yml@main
    with:
      version: ${{ inputs.version }}
    secrets:
      MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
      MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
      GPG_SECRET_KEY: ${{ secrets.GPG_SECRET_KEY }}
      GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```

### Release Script

```bash
# Dry run
python3 zhijun-io-release.py rose 0.1.0 --dry-run

# Actual release
python3 zhijun-io-release.py rose 0.1.0
```

## Setting Up Secrets

### Organization-Level Secrets (Shared)

Set these at the GitHub organization level (Settings > Secrets and variables > Actions):

| Secret | Description |
|--------|-------------|
| `MAVEN_USERNAME` | Sonatype Central Portal username |
| `MAVEN_PASSWORD` | Sonatype Central Portal token |
| `GPG_SECRET_KEY` | ASCII-armored GPG private key |
| `GPG_PASSPHRASE` | GPG key passphrase |

### Verifying Maven Central Credentials

1. **Web Portal**: Log in at https://central.sonatype.com with your credentials
2. **Test Deploy**: Try publishing a SNAPSHOT version to verify credentials work

### Getting GPG Key in ASCII Armor Format

```bash
# List your keys
gpg --list-secret-keys --keyid-format LONG

# Export private key (replace KEY_ID with your key ID)
gpg --armor --export-secret-keys KEY_ID > private-key.asc

# The contents of private-key.asc go into GPG_SECRET_KEY secret
```

## Examples

| Project | Path | Notes |
|---------|------|-------|
| Rose | [examples/rose/](examples/rose/) | Java 8, `${revision}`, `staged-ci` + `coverage` |

## Projects

| Project | Description | Workflow Type |
|---------|-------------|---------------|
| rose | Spring Boot 2.7 extension platform (`io.zhijun`) | Maven CI / Central release |

## Documentation

See [RELEASE.md](RELEASE.md) for detailed documentation on:
- Reusable workflow inputs and secrets
- Release script features
- Migration guides
- POM requirements
