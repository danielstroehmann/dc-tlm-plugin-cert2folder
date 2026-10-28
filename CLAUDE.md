# dc-cert2folder

<!-- BEGIN commons (managed by commons-init) -->
## Shared conventions (imported from stroehmi-claude-commons)
Managed block — do NOT edit the imported rules here. Edit the source repo
and run 'commons-sync'. Project-specific rules go BELOW the END marker.

@.claude/commons/agent-efficiency.md
@.claude/commons/code-style.md
@.claude/commons/deployment.md
@.claude/commons/dev-loop.md
@.claude/commons/git-workflow.md
@.claude/commons/security.md
@.claude/commons/testing.md
@.claude/commons/ui-ux.md
<!-- END commons -->

## Project-specific conventions

Plugin name: **cert2folder** (Maven artifact `cert2folder-tlm-plugin`), a DigiCert Trust Lifecycle Manager
(TLM) automation plugin that manages certificates in a directory on a remote host via SSH. Concept and open
decisions: `STATUS.md`. Everything learned while building the sibling plugin `dc-tlm-plugin-kemp`
(TLM runtime, sensor, UI behaviour, build and release): `KNOWLEDGE.md` - read it before designing or coding.

### Public repository

This repo is published on GitHub (`danielstroehmann/dc-tlm-plugin-cert2folder`). Nothing committed may point
to a concrete environment (own domains, hostnames, IPs, serial numbers, usernames, local paths).

- Docs, examples, defaults and tests use generic values only: `example.com`/`example.net`
  (RFC 2606) and `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` (RFC 5737).
- Everything used against a real system (test runbooks, results, certificates, keys, SSH credentials, notes)
  lives under `local/` - gitignored, never committed.
- Run `scripts/check-public.sh` before every commit; it must pass. Environment-specific
  strings to block go into the gitignored `local/public-denylist.txt`.

### Private project

The maintainer builds this privately. Commits use the repo-local git identity (the maintainer's GitHub noreply
address, set in `.git/config`), never a global employer identity; no employer affiliation, e-mail or branding in
repo content. The README states it is not an official DigiCert product.

### Documentation

- **Everything is English** - Markdown files, code comments, Javadoc, script headers and all UI texts
  (labels and help texts in `configuration.json`, error messages shown in TLM). Log and error messages are
  plain ASCII (the sensor log shows "?" for other characters). The plugin targets a global audience; this
  overrides the commons rule "UI texts German" for this repo. Chat with the maintainer stays German.
- Markdown files use upper-case base names (`SETUP.md`, `STATUS.md`); `.claude/commons/` is
  managed by commons-sync and keeps its names.
- The commons rules for web UIs (dev-loop overlay, PageSpeed, Docker deployment) do not apply: this is a
  plugin JAR without UI, run by the TLM sensor.

### Releases

- Releases (GitHub Releases with the plugin ZIP, `configuration.json`, `SHA256SUMS` and the
  `CHANGELOG.md` section as release notes) are built **only on the maintainer's explicit request**.
  Never create or push a version tag, or trigger a release, on your own. Process: `RELEASING.md`.
- Keep `CHANGELOG.md` -> `[Unreleased]` up to date with every user-visible change.
- Versions are strictly `x.x.x` (TLM accepts no other plugin version format) - no `-alpha`/`-rc` suffixes.
  Every `0.x.y` release is published as a GitHub pre-release; `1.0.0` only when the maintainer announces it.
- Every TLM upload costs about 10 minutes of scanning: hand over only complete, fully verified builds,
  checked live against a real target before the upload.
