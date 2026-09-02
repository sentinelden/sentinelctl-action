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
| `binary` | required | Path to the `.ipa`, `.app`, `.apk` or Mach-O |
| `severity-threshold` | `high` | Non-zero exit at this severity or above |
| `license-key` | empty | Optional. Without one it runs in preview mode |
| `output-dir` | `sentinel-reports` | Where reports are written |
| `sentinelctl-version` | `1.4.1` | Pin against the action major |

## Outputs

`sarif-path`, `sbom-path`, `finding-count`.

## Exit codes

`0` clean, `2` findings at or above your threshold. Reports are written and
uploaded as an artifact either way, because a failing gate is still a report
you want to read.

## What it does not do

- **No PDF reports.** PDFKit is macOS-only. Use the Markdown report.
- **No dynamic analysis.** Frida orchestration needs a device on a USB bus,
  which CI runners do not have. This is static analysis only.
- **Linux x86_64 only** today. `ubuntu-latest` is the tested runner. arm64
  runners will fail the architecture check with a clear message rather than
  downloading the wrong binary.

## Integrity

The action downloads a binary and executes it against your build artifact
inside your CI. It verifies the published SHA-256 before executing, and
**fails closed** if no checksum is published. Verify a download yourself with:

```bash
sha256sum -c sentinelctl-amd64.sha256
```

## Privacy

The audit runs entirely on your runner. Your binary is never uploaded
anywhere. Without a `license-key` the action makes no network call except to
download the CLI itself.

## Licence

Free to use, including in commercial CI. See [LICENSE](LICENSE). Your reports
and SBOMs are your property. No redistribution as a competing product.
