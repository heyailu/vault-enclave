# Ailu vault enclave

This repo is the public, attested record of what runs in Ailu's production vault: `tinfoil-config.yml`
(image by digest, settings, egress allowlist, secret names) and the release workflows. The vault's code
lives in `heyailu/ailu` (`app/vault`). **Merging a change to `tinfoil-config.yml` releases it to
production**, so never merge without asking the user at that moment.

## Security: red-team every change

Code review on pull requests (the Claude review workflow, `/pr-review`, or `/code-review` run by hand) must treat this section as required
checks and flag **security issues** at any confidence it can back with a concrete attack, not only certain
bugs. Review each change adversarially, as an attacker would: in the lines it changes, and in what it does to
what the enclave's attestation promises families. Each finding names the attacker, the steps, what they gain,
the file and lines, and a fix; mark anything that depends on Tinfoil's behaviour or context you can't verify
"needs human judgement". The same rules apply when writing changes.

- **Image.** The container image is pinned by `sha256` digest, never a tag, and comes from
  `ghcr.io/heyailu/`. A new digest should match an image built from a reviewed `heyailu/ailu` commit; say
  which one in the PR.
- **Egress.** Every host on the allowlist is a way for family data to leave the enclave. Flag any new host,
  wildcard or broadened entry, and ask what the vault sends there and whether it could carry keys or
  plaintext family data.
- **Secrets.** Secret values never appear in this repo, in env values or in workflow logs. Flag any new
  secret name (what is it, who holds it, does it let anyone outside the enclave decrypt or impersonate) and
  any secret moved from `secrets` into plain `env`.
- **Settings.** Flag env changes that weaken security: `RP_ID` or `APP_ORIGIN` widened to a domain an
  attacker could serve passkeys or requests from, debug or dev modes, disabled checks, or a database or
  schema that isn't production's.
- **Attestation.** Flag anything that changes what a release measures, lets the running vault differ from
  the latest release, adds a way to change the running vault without a reviewed release, or degrades
  `cvm-version`.
- **Workflows.** Flag changes to `.github/workflows/`: actions not pinned to a release, `pull_request_target`,
  new write permissions, `TINFOIL_ADMIN_KEY` or other secrets reachable from pull request code, and steps
  that update the production instance outside the reviewed release path.
