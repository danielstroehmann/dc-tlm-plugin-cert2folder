# NFR: Dev Loop & Feedback

> Binding rules. A deviation requires an explicit, reasoned proposal and the
> author's consent. Goal: fast innovation cycles — the author sees a new feature
> running within minutes and can point at exactly what to change.

## Local first

- **The primary loop is local.** Every project has **one command** that starts
  it locally with live reload / automatic rebuild: `scripts/dev-up` (e.g.
  `docker compose watch` or the stack's dev server behind it).
- **A feature is "visible" when it runs locally** and the agent has told the
  author the URL and where to look — not when it "should work".
- **Remote dev/test deployments are opt-in.** A deploy to a home-lab test system
  exists only when the project's own CLAUDE.md asks for it, in the form its
  architecture suggests. It is not a default.
- **Build identity is visible**: version, git short SHA and build time — in the
  UI footer of dev builds and via `/version` (or an about view) in all builds —
  so the author always knows which state they are looking at.
- **Seed / demo data**: the local dev start brings enough data that a new
  feature is visible immediately, without manual setup. Seeding runs only in dev
  mode and never touches existing data.
- The inner loop is not gated by long-running tests or reviews — see the release
  gate in [testing.md](testing.md).

## UI feedback overlay

The author gives UI and text feedback by pointing at the element in the running
app, not by describing it or annotating screenshots.

- **Every web UI includes a feedback overlay in local/dev builds.** It is
  **excluded from release builds at build time** (not merely hidden), and the
  endpoint that stores feedback exists only in dev mode.
- **Interaction:** a small floating button, the same on Mac and iPhone (no
  hotkey to remember) → pick an element → type a comment → save.
- **Each item records:** comment, element (stable selector and component name),
  its visible text and text key (see below), route/URL, viewport/device, build
  identity, a screenshot, timestamp, and `status` (`open` / `done`).
- **Storage:** one JSON file per item (plus its screenshot) under
  `./data/feedback/`, never committed.
- **Processing:** when the author says "work through the feedback", the agent
  takes all `open` items, implements them, and sets each to `done` with a
  one-line note of what changed. Unclear items are asked about, not guessed.
- Element picking and screenshots use an **established library** (no NIH, see
  [code-style.md](code-style.md)). The overlay looks and behaves the **same in
  every web app** (see [ui-ux.md](ui-ux.md)) — ideally one shared package.

## UI texts

- **All UI texts live in one central resource file per language** (e.g.
  `de.json`, or the stack's standard i18n mechanism) under **stable,
  descriptive keys** — no text literals in components.
- The author changes texts by editing that file directly or by saying
  "`key` → new text"; the feedback overlay records the key of a picked text.
- Texts are German with correct umlauts (see [ui-ux.md](ui-ux.md)).
