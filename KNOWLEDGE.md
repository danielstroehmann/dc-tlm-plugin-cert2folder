# Knowledge from the Kemp LoadMaster Plugin

What was learned while building and testing the sibling plugin
[dc-tlm-plugin-kemp](https://github.com/danielstroehmann/dc-tlm-plugin-kemp) (1.0.0, verified end to end in a TLM
tenant with a real sensor). Facts marked *verified* were observed on a real sensor or tenant; the rest comes from
the SDK, its examples or DigiCert's documentation.

## 1. Plugin runtime on the sensor

- Every action (`testConnection`, `refreshConfiguration`, `generateCsr`, `installCertificate`,
  `validateCertificate`) runs as a **separate JVM**. Nothing survives in memory between actions; state that must
  survive from `generateCsr` to `installCertificate` (the private key) has to be stored on disk.
- Input is the file `./plugin-info.json` in the working directory, not stdin (the SDK README is wrong there).
  The result is POSTed by the SDK to `TPM_COMMUNICATION_URL`. The process exit code is 0 even on failure.
- *Verified* format of `plugin-info.json`: stages are lower case (`config`, `run`). The `config` arguments contain
  `name` (the plugin name entered in TLM), `version`, `plugin_upgrade`, `baseRestApiUrl` and then the
  `config_attributes` **without** the prefix.
- *Verified*: **settings in the credential set `sensitive` arrive Base64-encoded** (not documented). The Kemp
  plugin 0.1.0 used the API key as delivered and got HTTP 401. Decode canonical Base64 of printable text and keep
  plain values as they are; trim whitespace and invisible characters (no-break space, zero-width space, BOM).
  Expect the same for passwords and private keys in this plugin.
- `plugin-meta.json` sets `PROCESS_TIMEOUT=30m` and `MAX_CHUNK=15`; the JAR is started with the sensor's Java
  (`TG_JAVA_HOME`). The warning "TPM_HOME environment variable not set" is harmless.
- Logs: plugin stdout/stderr go to the sensor's `tpm-wrapper.log` (DEBUG lines included); the job input is in
  `tpm-manager.log`. Both contain sensor secrets (license key, STOMP password, run tokens) and never go into a
  repository. *Verified*: non-ASCII characters appear as `?`, so all log and error messages are plain ASCII.
- Never log secrets. Useful for diagnosing credential problems: length, whether the value was Base64-encoded,
  positions of non-printable characters, a short SHA-256 prefix of the value as used.

## 2. Requests TLM sends (*verified*)

- `installCertificate`: `flowId`, `ipAddress`, `port` (a **string**), `virtualServerName` = `default/<alias>`,
  `managementIp` = null, `currentCertificateThumbprint` = SHA-256 of the replaced certificate, **upper case**
  hex, and `certificateLink` pointing to a ZIP on `one.<region>.digicert.com/mpki/api/v1/ts/artifact/...`.
  Further fields (keyStorage, redirectPort, idNamePattern, projectId, ...) can be ignored.
- `validateCertificate`: the serial as **lower-case hex with leading zero**; compare ignoring case, colons and
  leading zeros.
- `generateCsr`: subject DN as configured in the TLM profile (it contains the profile's O/L/ST/C and may contain
  an e-mail address), DNS names, key algorithm and size, signature algorithm.
- The SDK DTOs lack `ipAddress`, `port` and `virtualServerName`: use own request classes with
  `@JsonIgnoreProperties(ignoreUnknown = true)` and a local `ObjectMapper`; `PluginUtils.convertWrappedObject`
  fails for non-generic DTOs.
- The SDK's `DownloadCertificateUtil` disables TLS verification. The Kemp plugin downloads the package itself
  (`java.net.http`, JVM trust store, https only, no https-to-http redirects, size and entry limits, ZIP with
  `.cer/.crt/.pem/.p7b` or a single file).

## 3. How TLM shows the inventory (*verified*)

- Endpoints are IP based: `dataIp` + `port`, `alias`, `partition` (`default`), `sslState` 1,
  `portWithCert` = `<port>:<id1>,<id2>`; certificates carry `ipAndPort`, PEM, `sni`, `serverParam` (plugin's own
  identifier) and `domainName`.
- One certificate on several endpoints: TLM runs **one** flow for one endpoint; the plugin must update all uses.
- Several certificates on one endpoint (SNI): TLM sends the endpoint's alias; the **thumbprint selects** the
  certificate.
- **Endpoints without a certificate** appear under "Unsecured / Not configured", but TLM offers **no certificate
  request** for them (only "View audit trail"). The working solution was to report the endpoint's current
  (built-in) certificate: TLM then offers **"Renew"**, and on renewal the CN and SANs **can be changed** - the
  plugin turned that renewal into a first install.
- Identifiers in `serverParam` should be unique per distinct certificate; the same value for different PEMs may
  confuse TLM.
- The "Add connector" tile shows the plugin under **Appliances**, with the entered name in lower case; there is
  no display-name field in `configuration.json` or `plugin-meta.json`.

## 4. Uploading and updating the plugin

- *Integrations > Plugins > Add plugin*, type **Automation**, version **x.x.x only**, ZIP (max. 100 MB) plus
  `configuration.json`, "Verify and create". Needs the Automation feature and the "Plugin developer" role.
  DigiCert docs: "Add a plugin in Trust Lifecycle Manager".
- The scan takes **about 10 minutes** (status Pending, then Active). Hence: only upload complete builds, verified
  live against a real target before the upload.
- Updating: "Edit" the plugin, new version and ZIP. If `configuration.json` is unchanged, nothing else is needed.
- Connector: *Integrations > Connectors > Add connector*, category Appliances.

## 5. `configuration.json`

- Sections: `metadata`, `core_settings` (sensor select with `options_provider: "Sensors"`), `config_settings`,
  `additional_settings`, `credential_sets`.
- Field types seen: `input`, `password`, `select` (static `options` or `options_provider`, e.g. `Sensors`,
  `PamConnectors` for secrets managers), `multi_items_list`, `text` (static label). No multi-line input field is
  known.
- `conditional_group` (`{name, value, required}`) shows a field only for a given value of a select - used in the
  SDK example for "direct input" vs. "secrets manager"; suitable for "password" vs. "private key".
- Validation rules: `requiredRule`, `alphaNumericRule`, `customRegexRule` with `regex` and `error_message`.
- `credential_sets`: `sensitive` (encrypted at rest in TLM, delivered Base64-encoded, see section 1) and
  `unique` (fields that must be unique across connectors).
- Field names `config_attributes.<x>` must match the POJO field names exactly, otherwise the value silently
  arrives as null. A `@JsonAnySetter` that records only the **names** of unmapped settings helps diagnose this.

## 6. Security patterns that worked

- Key generation through the SDK's `CryptoPolicy` (RSA 2048-8192, EC P-256/384/521, SHA-256/384/512), DNS names
  validated (RFC 1123, no IP literals as DNS names).
- Private key between `generateCsr` and `installCertificate`: `CryptoPolicy.pbeWrapPrivateKey` (PBKDF2-HMAC-SHA256,
  600,000 iterations, AES-256-GCM) with a connector secret as password, in an owner-only directory (POSIX
  `0700`/`0600`, ownership check without following symlinks), `flowId` validated (`^[A-Za-z0-9-]{8,64}$`), deleted
  after install, purged after 30 days.
- Building the chain from the issued package: the leaf is the certificate matching the flow key; refuse CA
  certificates, expired or not-yet-valid ones and an EKU without serverAuth; upload only intermediates that
  verifiably sign the path, never roots.
- Idempotent installs: a retry after a completed install must succeed without changes (the key is gone by then).
- Check permissions and plan everything before the first write; roll back what was written if a later step fails.
- Never trust data from the target: re-encode certificates from the parsed X.509 before reporting them to TLM.

## 7. Build, release and repository

- SDK from the GitHub Pages Maven repo `https://digicert.github.io/tlm-plugins-sdk-dist` with
  `checksumPolicy=fail` (`settings.xml`). `scripts/verify-sdk.sh` pins the SHA-256 of the SDK jar **and** POM and
  runs right after dependency resolution, **before** any build or test executes SDK code (finding of the
  security review).
- CI on every push (tests only); the release pipeline runs only on a `vX.Y.Z` tag on a commit of `main`, checks
  the tag against the POM and the CHANGELOG, builds, attaches build provenance and publishes ZIP,
  `configuration.json` and `SHA256SUMS`. Actions pinned to commit SHAs, publish job without project code.
- The release ZIP is reproducible: the pipeline build was byte-identical to the local build.
- `scripts/check-public.sh` blocks private IPs, e-mail addresses, home paths, private keys and the patterns in the
  gitignored `local/public-denylist.txt`.
- A local runner that mirrors the sensor's input format (lower-case stages, Base64 for sensitive settings,
  name "Custom", extra settings) made local tests representative: `scripts/run-plugin-action.sh` in the Kemp repo,
  to be adapted here.

## 8. Lessons from the process

- Check assumptions live against the real target before an upload: the Kemp plugin's "built-in certificates are
  self-signed" assumption was wrong (they are issued by a vendor CA) and would have broken a whole feature.
- Independent test agent, reviewer agent and security review at the release gate found real defects each round.
- macOS has no `timeout` command, and in zsh `${2:+-flag $2}` is not word-split: both once produced a false
  "service down" result. Re-check before raising an alarm.
- Do not give the maintainer TLM UI instructions they did not ask for; they know TLM. Ask specific questions
  instead (e.g. whether a field can be edited on renewal).
