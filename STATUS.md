# Status: cert2folder Plugin for DigiCert TLM

As of: 2026-10-28 · Phase: concept, no plugin code yet

This document records what the plugin is meant to do, the assumed target environment, the proposed design and
the decisions still open. What was learned about TLM while building the sibling Kemp plugin is in
[KNOWLEDGE.md](KNOWLEDGE.md); read it first, several design points below depend on it.

---

## 1. Goal

A TLM **automation** plugin (SDK `com.digicert.tlm:plugin-sdk` 1.2, Java 17) that manages certificates as files
in a directory on a remote host, reached via SSH:

- **One connector instance = one SSH target**, with its own credentials: username and password (first version),
  later also username and an authorized private key (D4).
- The plugin creates the configured **base directory** if needed (e.g. `/etc/digicert`) and manages certificates
  in it **like certbot does in `/etc/letsencrypt`** (layout in section 3).
- **Rollout of new certificates** and **renewal**: TLM issues, the plugin writes the files.
- **Existing certificates** in the base directory are read during the inventory and brought under automation.
- **Post-deployment command** per connector instance, e.g. `docker restart web`, run on the remote host after
  every deployment. Before it runs, environment variables point at the deployed files, so the command can use
  them (section 5).

Typical use: services on a Linux host or in Docker containers that read their certificate from files
(nginx, HAProxy, Traefik, Postfix, any container with a bind mount) and are restarted or reloaded afterwards.

## 2. Connector settings (draft)

| Setting | Required | Notes |
|---|---|---|
| Sensor | yes | standard core setting |
| Host | yes | hostname or IP of the SSH target |
| Port | yes | default 22 |
| Username | yes | |
| Password | yes | credential set `sensitive` (delivered Base64-encoded, see KNOWLEDGE.md) |
| Base directory | yes | default `/etc/digicert`; absolute path, restricted character set, no `..` |
| Output format | yes | select: **PEM** (default) or **PKCS#12** (D10); no other formats for now |
| PKCS#12 password | if format = PKCS#12 | credential set `sensitive`; shown via `conditional_group`; protects the `.p12` file |
| Post-deployment command | no | run after every deployment, see section 5 |
| Host key fingerprint | no | `SHA256:...` of the target's SSH host key; if empty, trust on first use (D3) |

Private-key authentication (method select via `conditional_group`, key and passphrase in the credential set
`sensitive`) follows in a later version (D4).

The host (plus port) goes into the `unique` credential set, as in the Kemp plugin, unless several connectors per
host with different base directories should be allowed (then host + base directory).

## 3. Directory layout (like certbot)

```
<base>/
  archive/<name>/cert1.pem  chain1.pem  fullchain1.pem  privkey1.pem    # one numbered set per deployment
                 cert2.pem  chain2.pem  fullchain2.pem  privkey2.pem
  live/<name>/cert.pem       -> ../../archive/<name>/cert2.pem          # relative symlinks to the newest set
              chain.pem      -> ../../archive/<name>/chain2.pem
              fullchain.pem  -> ../../archive/<name>/fullchain2.pem
              privkey.pem    -> ../../archive/<name>/privkey2.pem
```

- `<name>` (the "lineage") is derived from the certificate's first DNS name, as certbot does; a wildcard
  `*.example.com` becomes `example.com`. On collision a suffix `-0001`, `-0002`, ... is added (certbot scheme).
  Allowed characters: `[A-Za-z0-9.-]`, so names are safe in paths and in shell variables.
- A deployment writes the new numbered set to temporary names, renames them into place and then switches the
  symlinks atomically (new symlink under a temporary name, then rename over the old one). A service reading
  `live/<name>/` never sees a half-written set.
- Permissions as certbot: `archive/` and `live/` `0700`, `privkey*.pem` `0600`, others `0644` (decision D6).
- Old numbered sets are kept (certbot keeps them too); cleanup is a later option.
- With output format **PKCS#12**, each set additionally contains `keystore<N>.p12` (certificate, chain and key,
  protected with the configured password; `0600`), linked as `live/<name>/keystore.p12`. The PEM files are always
  written: they are the basis of the layout and of the inventory, which reads `cert.pem`.
- `<base>/.tlm/placeholder.pem`: the self-signed placeholder certificate for new rollouts (D2), certificate only,
  its key is discarded right after creation.
- certbot's `renewal/<name>.conf` is **not** written: it would make certbot treat the lineage as its own.

## 4. TLM actions (proposed)

| Action | What the plugin does on the target |
|---|---|
| `testConnection` | SSH login, check that the base directory exists or can be created and is writable, report the remote user and whether the post-deployment command is set |
| `refreshConfiguration` | Read `live/*/cert.pem`, report one certificate per lineage (mapping: D1) and the placeholder for new certificates (D2) |
| `generateCsr` | Key and CSR **on the sensor** (D5); no SSH connection needed |
| `installCertificate` | Resolve by thumbprint first: an existing lineage is renewed (next numbered set), the placeholder creates a new lineage (D2). Then run the post-deployment command |
| `validateCertificate` | Find the lineage whose `live/<name>/cert.pem` has the serial (on any endpoint of the host, including the placeholder) and return it |

## 5. Post-deployment command

- Configured per connector instance; empty means no command.
- Runs over the SSH exec channel as the connector's user, after the symlinks have been switched, once per
  deployment (renewal or new lineage).
- **Environment variables** are set by prefixing the command with `export` statements, not via SSH `setenv`
  (sshd accepts only `AcceptEnv` names, usually just `LANG`/`LC_*`):

  | Variable | Value |
  |---|---|
  | `TLM_CERT_FORMAT` | `PEM` or `PKCS12`: tells the command which of the variables below apply |
  | `TLM_CERT_NAME` | lineage name |
  | `TLM_LINEAGE_DIR` | `<base>/live/<name>` |
  | `TLM_CERT_PATH` | `<base>/live/<name>/cert.pem` |
  | `TLM_CHAIN_PATH` | `<base>/live/<name>/chain.pem` |
  | `TLM_FULLCHAIN_PATH` | `<base>/live/<name>/fullchain.pem` |
  | `TLM_KEY_PATH` | `<base>/live/<name>/privkey.pem` |
  | `TLM_P12_PATH` | only with `PKCS12`: `<base>/live/<name>/keystore.p12` |
  | `TLM_P12_PASSWORD` | only with `PKCS12`: the configured PKCS#12 password |
  | `TLM_DOMAINS` | DNS names of the certificate, space-separated |
  | `RENEWED_LINEAGE`, `RENEWED_DOMAINS` | same as certbot sets for deploy hooks, so existing certbot hook scripts work unchanged |

  For `PEM` the command typically needs `TLM_FULLCHAIN_PATH` and `TLM_KEY_PATH`, for `PKCS12` `TLM_P12_PATH` and
  `TLM_P12_PASSWORD`; the PEM paths are set in both cases.
  Path values contain only the restricted character sets of section 2/3 and are single-quoted, so they cannot
  break out of the assignment. The PKCS#12 password is quoted with every `'` escaped; it is never logged, and the
  command line it is part of is not logged either.
- Exit code and output: decision D7. A time limit applies (proposal: 5 minutes; the sensor's process limit is
  30 minutes).
- The command is trusted input from the TLM administrator. Whoever may edit the connector can run commands on
  the target as the connector's user; this is the intended capability and is documented in SETUP.md later.

## 6. Assumed target environment

- Linux or another POSIX system with OpenSSH server, SFTP subsystem enabled, `sh`, `ln`, `mv`, `mkdir`.
- A dedicated user for the plugin is recommended. Writing to `/etc/digicert` needs either root or a base
  directory created once and owned by that user; `docker restart` needs membership in the `docker` group (or
  root). Whether `sudo` support is needed: D8.
- The sensor reaches the target via SSH (TCP 22 or the configured port).

## 7. Decisions

### Open

None at the moment.

### Made

| Date | Decision |
|---|---|
| 2026-10-28 | **D1, mapping to TLM endpoints** (technical, decided by the developer): TLM's automation inventory needs every certificate attached to an endpoint (IP, port, alias), otherwise TLM does not show it or offer "Renew". The plugin reports one endpoint per lineage: the host's IP and the SSH port, alias = lineage name. If TLM does not keep several endpoints with the same IP and port apart by alias, fallback: one endpoint per host with all lineages as SNI-style entries, selected by thumbprint (verified with the Kemp plugin). For the user this is invisible: certificates are only files in the directory. |
| 2026-10-28 | **D6, permissions as certbot**: `archive/` and `live/` `0700`, keys `0600`, other files `0644`. |
| 2026-10-28 | **D7, failing post-deployment command fails the TLM action**: TLM shows the (truncated) output; the deployed files stay, no rollback. |
| 2026-10-28 | **D8, no privilege escalation in 0.1**: a plain user with a prepared base directory (and e.g. `docker` group membership for the command); `sudo` not supported for now. |
| 2026-10-28 | **D9, SSH library Apache MINA SSHD** (developer's choice): pure Java, so it runs the same on Windows and Linux sensors; Apache project with SFTP, modern algorithms and host key verification hooks; doubles as embedded SSH server in the tests (no NIH, one library). |
| 2026-10-28 | **D10, output format as a select in the connector: PEM (default) or PKCS#12**; no other formats for now. PEM files are always written; PKCS#12 adds `keystore.p12` protected with a password configured in the connector. `TLM_CERT_FORMAT` tells the post-deployment command which variables apply: PEM -> fullchain and key path, PKCS#12 -> keystore path and password. |
| 2026-10-28 | **D2, new certificates via a placeholder endpoint.** Per host the inventory reports one extra endpoint (alias `new-certificate`) carrying a self-signed placeholder certificate. "Renew" on it, with the desired names, creates a new lineage; the placeholder stays, so any number of certificates can be rolled out to one host. The placeholder is created once and stored as certificate only in `<base>/.tlm/placeholder.pem`, so its fingerprint stays stable (a new one per inventory would add a new certificate to TLM's inventory every time). `installCertificate` resolves by thumbprint before endpoint, because TLM may keep a new certificate associated with the placeholder endpoint and renew it there. To verify in the tenant: how TLM shows a certificate installed via the placeholder and on which endpoint it schedules renewals. Reason: TLM offers no certificate request for plugin endpoints without a certificate, but offers "Renew" with editable names (verified with the Kemp plugin). |
| 2026-10-28 | **D3, SSH host key verification**: optional fingerprint setting; if empty, trust on first use with the fingerprint stored on the sensor (owner-only, like the flow keys) and logged; a changed host key fails every action before any credential or key is sent. Reason: without it, whoever impersonates the target receives the password and every deployed private key. |
| 2026-10-28 | **D4, first development version authenticates with username and password only.** Private-key authentication (and the single-line input question of TLM's password field) follows later. |
| 2026-10-28 | **D5, key and CSR are generated on the sensor**, not on the target. Reasons: (1) generating on the target would require OpenSSL there, an extra prerequisite on every host; (2) `generateCsr` then needs no SSH connection, so an unreachable target fails only at installation, as with the Kemp plugin; (3) the code path (SDK `CryptoPolicy`, encrypted key store between CSR and install, deletion after install) is proven in the Kemp plugin. The key travels to the target only inside the authenticated, encrypted SSH session, which is why host key verification (D3) is mandatory. Accepted trade-off: the key exists briefly outside the target host. |
| 2026-10-28 | **D11 / Let's Encrypt discovery deferred**: the read-only discovery of certificates in `/etc/letsencrypt` is left out for now and may come back later; nothing in the design depends on it. |
| 2026-10-28 | Directory name `dc-cert2folder`, plugin name `cert2folder`, public GitHub repository `danielstroehmann/dc-tlm-plugin-cert2folder` from day one, MIT license. Structure and conventions as in `dc-tlm-plugin-kemp`. |
| 2026-10-28 | Credentials per connector instance: username + password; an authorized private key later (D4). |
| 2026-10-28 | Layout like certbot (`archive/`, `live/`, numbered sets, symlinks); a post-deployment command per instance with environment variables pointing at the deployed files. |

## 8. Next steps

- [x] All design decisions made (2026-10-28) (or confirms the proposals)
- [ ] Test target: Docker container with OpenSSH locally; a real host in the home lab (details in `local/`)
- [ ] First version 0.1.0: plugin code, tests by the separate test agent, review, security review
- [ ] SETUP.md for target preparation (user, directory, SSH key, `docker` group)
- [ ] Upload to the TLM tenant, verify D1 and D2 behaviour
