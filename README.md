# Kora Release Distribution

This repository hosts public Kora release distribution artifacts. Source code remains in the private `raw-labs/kora-platform` repository.

## Self-Managed Deploy Bundle

Use the GitHub Release assets for the version you want to install:

- `kora-platform-deploy-<tag>.tar.gz`
- `kora-platform-deploy-<tag>.tar.gz.sha256`

Download both files from the same release, verify the checksum, extract the archive, and run `./koractl install`.

## Claude Code Plugins

Add this repository as a Claude Code plugin marketplace, then install the Kora skill packages:

```sh
claude plugin marketplace add raw-labs/kora-releases
claude plugin install kora-authoring@kora
claude plugin install kora-operations@kora
```

For a pinned install, include the release tag when adding the marketplace:

```sh
claude plugin marketplace add raw-labs/kora-releases@<tag>
```

## Codex Plugins

Add this repository as a Codex plugin marketplace. Pin a release tag when you need reproducible installs:

```sh
codex plugin marketplace add raw-labs/kora-releases --ref <tag>
codex plugin add kora-authoring@kora
codex plugin add kora-operations@kora
```

## Versions

Final tags such as `0.4.0` are stable release channels. Release-candidate tags such as `0.4.0-rc.1` are prerelease channels for validation before a final release.

Use final tags for normal installs. Use RC tags only when testing a release candidate. The default branch tracks final releases; the `rc` branch tracks release-candidate plugin output. Consumers that need reproducibility should install from a tag instead of a moving branch.
