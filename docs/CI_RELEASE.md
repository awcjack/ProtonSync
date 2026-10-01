# GitHub builds and signed releases

## Workflows

- **Android CI**: pull requests, selected branch pushes, and manual runs. Runs JUnit tests and builds an **unsigned release APK**; retains APK/test artifacts for seven days. No signing secrets are used.
- **Android Release**: pushes of tags matching `v*`, or manual dispatch on an existing tag. Only stable `vMAJOR.MINOR.PATCH` tags are accepted. Builds/tests without secrets, signs on a separate runner, verifies signature/certificate/alignment/non-debuggable status/backup policy, and publishes the APK, `SHA256SUMS`, certificate information and build metadata to a GitHub Release.
- The signing job does not check out source or execute Gradle. Only the publish job has `contents: write`. Action versions are pinned to commit hashes.
- Builds use JDK 17, Android platform 35 and Build Tools 34.0.0; signing uses Build Tools 35.0.0. GitHub-hosted Ubuntu runners provide `sdkmanager` and accepted Android SDK licenses.

These workflows do not fix outstanding application audit findings. Publishing/signing is not a security certification. Device testing is still required.

The CI-only commit does not include the local application fixes or regression tests. Until those are committed, test tasks may have no tests to run. The signing job deliberately rejects APKs with backup enabled: the original upstream manifest fails that check. Commit and review the backup hardening before creating a release tag; do not bypass the check.

## One-time repository setup

Push this repository/workflow code to **your own GitHub repository or fork** before creating a new release tag. This checkout uses `https://github.com/awcjack/ProtonSync.git` as `origin`.

1. In **Settings → Environments**, create an environment named **`release`**.
2. Enable **required reviewers** (where supported), disallow self-approval where appropriate, and restrict deployment tags to trusted release tags such as `v*`.
3. Protect release tag creation/deletion using repository rulesets. Only trusted maintainers should be able to create release tags or change release workflows. A tag can point to any commit; its name alone does not establish trust. Review the tagged commit before approving signing.
4. Add the secrets and variable below **to that environment**, not to a pull-request workflow.

### Environment secrets

| Secret | Value |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | Base64 encoding of your existing `.p12`/keystore file |
| `ANDROID_KEYSTORE_PASSWORD` | Keystore password |
| `ANDROID_KEY_ALIAS` | Alias, e.g. `protonsync` |
| `ANDROID_KEY_PASSWORD` | Private-key password; optional when identical to the keystore password |

### Environment variable

| Variable | Value |
| --- | --- |
| `ANDROID_SIGNING_CERT_SHA256` | SHA-256 fingerprint of your signing **certificate**, not of an APK; colon-separated or plain hexadecimal accepted |

Use the **same key you used to sign your installed modified APK** so future updates can install over it. Do not generate a different key for every release. Keep an offline backup of the key and its password. The upstream developer's APK and earlier debug APKs have different signing identities and cannot be updated in place with your key.

### Configure with GitHub CLI (Linux)

First create/protect the environment in GitHub's UI. Substitute your own repository:

```bash
export GH_REPO=YOUR_ACCOUNT/ProtonSync

# Upload base64 directly without printing the private key into terminal output.
base64 -w 0 /workspace/protonsync-release.p12 | \
  gh secret set ANDROID_KEYSTORE_BASE64 --env release --repo "$GH_REPO"

# Enter secrets using gh's hidden interactive input. Do not put passwords in shell history.
gh secret set ANDROID_KEYSTORE_PASSWORD --env release --repo "$GH_REPO"
gh secret set ANDROID_KEY_PASSWORD --env release --repo "$GH_REPO"
gh secret set ANDROID_KEY_ALIAS --env release --repo "$GH_REPO"

# Display your public certificate fingerprint; keytool prompts for the password.
keytool -list -v -keystore /workspace/protonsync-release.p12 -alias protonsync

# Paste the SHA256 certificate fingerprint, not its SHA1 fingerprint.
gh variable set ANDROID_SIGNING_CERT_SHA256 --env release --repo "$GH_REPO"
```

On macOS, use `base64 < /path/to/key.p12` instead of `base64 -w 0 ...`. Never commit/upload the keystore, password files or decoded secrets as workflow artifacts. Base64 is **not encryption**; protect the GitHub secret like the original key.

## Create a release

After committing all intended changes (including new source files and workflows) and pushing to your repository:

```bash
git tag -a v1.1.0 -m 'ProtonSync v1.1.0'
git push origin v1.1.0
```

Use a new tag; do not move published tags. In GitHub Actions, approve the `release` environment deployment after reviewing the tagged commit. The workflow then automatically publishes the release; this is **not a draft**.

Outputs:

- `ProtonSync-1.1.0.apk` — signed, installable release build
- `SHA256SUMS` — APK SHA-256 checksum
- `signing-certificate.txt` — signature-verification output and certificate fingerprints (public)
- `build-info.txt` — source commit, app version/version code and workflow run link

The workflow deliberately refuses to overwrite an existing GitHub Release. For a failed run, use **Re-run failed jobs**; after publication, issue a new version/tag rather than replacing the assets.

## Versioning

The app accepts `-PreleaseVersionName` and `-PreleaseVersionCode`. Local builds retain the original defaults unless these are supplied.

Release `versionName` comes from the tag (`v1.1.0` → `1.1.0`). `versionCode` is **1000 + the Android Release workflow run number**. Re-running a job preserves its run number. This exceeds the original/local default code 1.

If moving repositories, deleting/recreating the workflow, or having already distributed higher version codes, adjust the base in `android-release.yml` so every newly installed update has a greater code than previously distributed builds. Dispatching old tags produces old source with a new run number; avoid doing so for distribution. This workflow accepts stable tags only, not prerelease suffixes.

## Verification and limitations

```bash
sha256sum --check SHA256SUMS
apksigner verify --verbose --print-certs ProtonSync-1.1.0.apk
```

Compare the certificate SHA-256 to your independently recorded fingerprint. The pipeline checks that fingerprint before publishing. Debug network logging follows `BuildConfig.DEBUG`, which is false for release builds; debug artifacts must not be distributed as production builds.

There are currently upstream Kotlin/D8 metadata warnings. The build can succeed despite those warnings. Dependencies/toolchains are not a fully reproducible locked supply chain, and native-library page-size/device compatibility still needs runtime testing. A CI checksum identifies the released artifact; it does not establish equivalence to an independently rebuilt binary.
