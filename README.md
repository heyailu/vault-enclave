# Ailu vault enclave

This repository is the public record of what runs inside **Ailu's vault**: the part of
[Ailu](https://heyailu.com) that holds each family's encryption key while their assistant is
working, and the only part that ever sees it.

The vault runs in a [Tinfoil Container](https://docs.tinfoil.sh/containers/overview), a sealed
hardware enclave. The enclave can prove which code it is running, and this repository is what that
proof is checked against. Nothing else about Ailu needs to be public for that check to work.

## What's here

| File | What it is |
|---|---|
| `tinfoil-config.yml` | Exactly what runs in the enclave: the container image (pinned by digest), its settings, and the only hosts it can talk to |
| `.github/workflows/auto-release.yml` | When a change to `tinfoil-config.yml` merges, picks the next version and starts a release |
| `.github/workflows/tinfoil-release.yml` | Tags a release. Unchanged from Tinfoil's template |
| `.github/workflows/tinfoil-release-publish.yml` | Measures the configuration, signs the measurement, and publishes the release (Tinfoil's template), then updates the running vault to it |

There is no source code and there are no secrets here. Secret values are delivered to the enclave by
Tinfoil at runtime; the config only names them.

## What a release means

Each GitHub release of this repository is a signed **measurement** of `tinfoil-config.yml`: a
fingerprint covering the image, every setting and the network allowlist. The signature is recorded
in the public [Sigstore](https://www.sigstore.dev) transparency log, so releases can't be quietly
rewritten.

When the Ailu app, or anyone else, connects to the vault, the enclave's hardware proves which
measurement it is running. If it matches a release here, the vault is running exactly what that
release describes, and nothing else.

## Checking a running vault yourself

With [Tinfoil's CLI](https://docs.tinfoil.sh/containers/cli):

```sh
tinfoil http get -e <vault host> -r heyailu/vault-enclave@<version> https://<vault host>/healthz
```

The CLI verifies the enclave's attestation against the named release and pins its TLS identity before
sending the request. Plain `curl` only checks that the vault is up; it doesn't verify anything.

## Changes

Every change to `tinfoil-config.yml` goes through a reviewed pull request before it can be released.
Releases are listed on this repository's
[Releases](https://github.com/heyailu/vault-enclave/releases) page.

Merging that pull request is the decision to release. After it merges:

1. `auto-release.yml` picks the next version (a minor bump) and starts the Tinfoil Release.
2. The release is measured, signed, and published, but not yet marked "latest".
3. The publish workflow updates the `app` instance to the new release. The new version boots
   alongside the old one, and traffic switches once it's running. Tinfoil then marks the release
   "latest".

The app checks the vault against the latest release. A new release becomes latest only once it's
the version serving, so publishing one never makes the running vault fail verification.

The update uses a Tinfoil organisation admin key, stored as the `TINFOIL_ADMIN_KEY` secret of
this repository's `production` environment. If the secret isn't set, the release is still published
and the run warns that the instance needs updating by hand. An update that would need downtime
(replacing the instance rather than blue/green) stops and fails instead of going ahead.
