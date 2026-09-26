# SentinelDen Studio Audit, GitHub Action

Audits your iOS or Android build on every pull request and surfaces the
findings in GitHub's own Security tab.

```yaml
name: Security audit
on: [pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    permissions:
      security-events: write   # required to upload SARIF
    steps:
      - uses: actions/checkout@v4

      # ... your normal build steps, producing an .ipa / .app / .apk ...

      - uses: sentinelden/sentinelctl-action@v1
        id: audit
        with:
          binary: ./build/MyApp.ipa
          severity-threshold: high

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: ${{ steps.audit.outputs.sarif-path }}
```

That is the whole integration. Findings appear alongside CodeQL's in
**Security → Code scanning**, annotated on the diff.

## What it does

Runs the same audit engine as the macOS SentinelDen Studio app: Mach-O and
APK/DEX parsing, MASVS control coverage, a CycloneDX SBOM, secret and
crypto-misuse detection, and a policy gate that can fail the build.

## Inputs

| Input | Default | Notes |
|---|---|---|
| `binary` | required | Path to the `.ipa`, `.app`, `.apk`, `.aab` or Mach-O |
| `severity-threshold` | `high` | Non-zero exit at this severity or above |
| `rule-packs` | empty | Signed rule packs exported from Studio (paths, comma- or newline-separated). Needs sentinelctl 1.7.0 or newer |
| `rule-pack-keys` | empty | Base64 Ed25519 keys a pack must be signed by. With a key set, any other pack is refused; without one, the action warns |
| `license-key` | empty | Accepted but not enforced: the action is free during early access |
| `output-dir` | `sentinel-reports` | Where reports are written |
| `artifact-name` | `sentinel-reports` | Name of the uploaded artifact. Must be unique per workflow run, so set it when you audit several binaries or use a matrix |
| `sentinelctl-version` | `1.7.1` | Pin against the action major |

### Your own rules

Export a rule pack from Studio, commit it, and pin the key it was signed with,
so a pack signed by anyone else fails the job instead of loading:

```yaml
      - uses: sentinelden/sentinelctl-action@v1
        with:
          binary: ./build/MyApp.ipa
          rule-packs: ./security/our-rules.sentinelpack.json
          rule-pack-keys: ${{ vars.SENTINEL_RULE_PACK_KEY }}
```

## Outputs

`sarif-path`, `sbom-path`, `finding-count`.

## Exit codes

`0` clean, `2` findings at or above your threshold, `1` bad input such as a
missing binary or a rule pack that fails verification. Reports are written and
uploaded as an artifact either way, because a failing gate is still a report
you want to read.

## What it does not do

- **No PDF reports.** PDFKit is macOS-only. Use the Markdown report.
- **No dynamic analysis.** Frida orchestration needs a device on a USB bus,
  which CI runners do not have. This is static analysis only.

## Runners

- **Linux, x86_64 or arm64.** The action picks `sentinelctl-amd64` or
  `sentinelctl-arm64` from `uname -m`. `ubuntu-latest` is the tested runner.
  Releases up to and including cli-v1.6.0 are x86_64 only, so pin 1.7.0 or
  newer on arm64. On a macOS runner the action stops with an error; install
  the CLI there with `brew install sentinelden/tap/sentinelctl`.
- **No glibc requirement.** From 1.7.0 the binaries are fully static, so RHEL 9
  and Amazon Linux 2023 runners work as well as Ubuntu. (1.6.0 and earlier
  need glibc 2.35 or newer.) The action's own steps need `bash`, `curl`,
  `sha256sum` and `sort -V`; a minimal image such as Alpine needs
  `apk add bash curl coreutils unzip` first.
- **`unzip`** at `/usr/bin/unzip` for `.ipa`, `.apk` and `.aab` targets.
  GitHub-hosted Ubuntu runners have it; a minimal self-hosted image may not.

## Integrity

The action downloads a binary and executes it against your build artifact
inside your CI. Before executing it, it checks the SHA-256 against the
checksum published beside the binary and against a digest committed in this
action for that version and architecture, and it **fails closed** if either
check fails or no checksum is published. Verify a download yourself with:

```bash
sha256sum -c sentinelctl-amd64.sha256
```

## Privacy

The audit runs entirely on your runner. Your binary is never uploaded
anywhere. The only network call the action makes is downloading the CLI
itself; a `license-key` is not sent anywhere.

## Licence

Free to use, including in commercial CI. See [LICENSE](LICENSE). Your reports
and SBOMs are your property. No redistribution as a competing product.
