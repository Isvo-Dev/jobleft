# Wiki: apps (server, ui, extension, shell), scripts, CI, evals, qa

Navigation notes for changing code. Contracts, rules and routes live in `docs/INTERFACES.md`; this page only points.
Paths are relative to the repo root. `file:N` = line N (checked 2026-09-29).

Don't duplicate, read instead: INTERFACES §2 (data folder), §4 (env vars), §5 (app:up / shell contract),
§6.1 (security rules), §6.4 (all 97 routes, generated), §7 (extension protocol), §8 app sections;
`apps/server/README.md` (§3 start, §9 stand-ins, §13 interim, §14 tests); `apps/ui/README.md` (demo mock, probes);
`apps/extension/README.md`; `apps/shell/README.md` (bundle layout, Windows); `docs/outcomes/{server,ui,extension,i-shell}.md`
(acceptance MUSTs); `docs/BUILD-REPORT.md` (what exists, gaps).

## 0. What is live in `apps/server` (965 tracked files)

| Path | Files | Status |
|---|---|---|
| `apps/server/src/` | 49 | **Live**: the server |
| `apps/server/test/` | 22 | Live tests (`node --test`) |
| `apps/server/ui-fallback/` | 3 | Live: the page served at `/` when no built UI exists |
| `apps/server/scripts/` | 3 | Dev tools: `mock-servers.ts` (stand-in boards/AI/publik), `mocks.ts`, `jobsync-tests.sh` (bash) |
| `apps/server/jobsync/` | **885** | **Vendored reference only.** Patched fork of jobsync (MIT, `527333e`, Next.js 15 + Prisma + Tailwind + shadcn + Nivo charts). Not in the pnpm workspace, not type-checked (`apps/server/tsconfig.json` includes only src/test/scripts), not packed (`apps/shell/scripts/pack.ts:30` skips it), never served. Port source only: `apps/server/jobsync/JOBLEFT-FORK.md` |

jobsync's Nivo chart components (reference for visualisations): `apps/server/jobsync/src/components/dashboard/{WeeklyBarChartToggle,JobsActivityCard,ActivityCalendar}.tsx`.

## 1. Server (`@jobleft/server`, plain `node:http` + `node:sqlite`, zero third-party deps of its own)

### Boot sequence
1. `apps/server/src/main.ts:28` parse `--home/--port` → `:45` `resolveHome()` → `:47-48` TMPDIR/SQLITE_TMPDIR = `<home>/tmp` → `:49` launch token (env or new), then deleted from env `:51`.
2. `:75` `startServer()` → `apps/server/src/server.ts:113`:
   - `:116` `ensureHome` (folders) · `:120` `acquireLock` (`run/server.lock`, one server per folder) · `:125` finish/undo interrupted restore · `:127` empty `tmp/`
   - `:129` secret store (`services/secrets.ts:98`) · `:132` UI dir = `JOBLEFT_UI_DIR` ?? `apps/ui/dist` (if it exists at start) ?? fallback
   - `:141` `app.open()` → `new AppData` (`apps/server/src/app.ts:78`): `:80` `openAndMigrate` (`db/open.ts:94`; server migrations `db/schema.ts:24`) → `:84` second connection via crawler `Store` → services (profile, jobs, tracker, network, AI facade, resumes, boards, feed)
   - `:149` `StaticSite` · `:155` `createServer` → `dispatch` `:165`
   - `:395` listen `127.0.0.1` first free of 47821-47830 (`:89`, `:93`) · `:402` write `run/server.json` · `:406` warm FTS
   - `:416` `startBackground` (`app.ts:192`): reminders every 30 s (`app.ts:196-197`) + crawl scheduler `interim/boards.ts:323` (catch-up 5 s after start, tick every 60 s, runs when `crawl.intervalHours` elapsed). Crawls run in a worker thread (`interim/boards.ts:256` → `interim/crawl-worker.ts`).
   - `:436` parent-pid watch (shell gone → exit) · `:452` shutdown hook for `POST /api/v1/shutdown`
3. Restore / delete-all swap the whole `AppData` while listening: `App.swap` `app.ts:289` (requests answer 503 `not_ready` meanwhile).

### Routes: registration and dispatch
- Route table = data, not code: `LOCAL_API` in `packages/contracts/src/api.ts:197` (method, path, auth, body/query/response schemas). `matchRoute` `api.ts:373`.
- Handlers: `HANDLERS` in `apps/server/src/routes.ts:145`, typed `{ [K in RouteName]: Handler<K> }` → a route without a handler does not compile. Handler returns `{json}` | `{file}` | `{sse}` (`routes.ts:47`).
- Where handler logic lives: `apps/server/src/interim/*` (INTERIM stand-ins, `srv_*` tables), `integ/*` (real packages: ai-engine, assistant, resume, match), `services/*` (backup, pairing, extension, kv, notifications), `core/*` (feed ranking, seeding).
- Dispatch order in `server.ts` (single place, INTERFACES §6.1): dup Host `:179` → Host allowlist `:185` → dot segments/backslash `:197` → Origin `:210-217` → non-`/api/` = static files `:219-224` → `Sec-Fetch-Site` cross-site `:227` → OPTIONS only for paired extension `:233` → `matchRoute` `:248` → devOnly `:252` → token-in-query refused `:255` → auth `launch`/`pairing`/`pair` `:262-278` → media type `:285-297` → maintenance/read-only `:299-311` → body + query schema validation `:313-351` → per-client turnstile (`http/turnstile.ts`, 128 waiting max → 429) `:356` → handler `:371` → dev-mode response contract check `:381`.

### Security gate (`apps/server/src/http/gate.ts`)
- Launch token: 32 random bytes base64url (`home.ts:43`), compared as sha256 in constant time (`gate.ts:21`), header `x-jobleft-token`. UI gets it in the URL fragment `/#token=...`.
- Host must be exactly `127.0.0.1:<port>` or `localhost:<port>` (`gate.ts:26`). Origin: none, own origin, or `chrome-extension://<32 a-p>` (`gate.ts:14`, `:32`).
- Pairing routes: `x-jobleft-pairing` + matching extension Origin, token hash in table `pairings` (`services/pairing.ts`).

### UI serving
- `apps/server/src/http/static.ts`: real files only, `/` = `index.html`, **no SPA fallback** (`static.ts:70-73`), `index.html` no-store.
- CSP `static.ts:38`: `script-src 'self'` (no inline/eval), `style-src 'self' 'unsafe-inline'`, `connect-src 'self'`, `font-src 'self' data:`, `frame-ancestors 'none'`.
- Fallback page `apps/server/ui-fallback/` (`server.ts:73`, `:149`).

### Data folder (`homeLayout` `home.ts:21`; full table INTERFACES §2)
| OS | Default | Code |
|---|---|---|
| Windows | `%APPDATA%\jobleft` (installed app too) | `home.ts:16`, `apps/shell/src-tauri/src/lib.rs:60` |
| macOS | `~/Library/Application Support/jobleft` | `home.ts:15` |
| other | `$XDG_DATA_HOME/jobleft` or `~/.local/share/jobleft` | `home.ts:17` |
| dev (`pnpm app:up`) | `<repo>/.jobleft-dev` (git-ignored) | `scripts/app-up.ts:29` |

Inside: `data/jobleft.db`, `files/`, `logs/` (`server.log` in dev, `sidecar.log` from the shell), `run/server.json` (pid, port, token), `run/server.lock`, `tmp/`.

## 2. UI (`@jobleft/ui`)

### Stack
| Thing | Choice | Where |
|---|---|---|
| Build | Vite 7.3.6, no React plugin (esbuild JSX), `base: './'`, out `apps/ui/dist` | `apps/ui/vite.config.ts` |
| UI lib | React 18.3.1, **Ant Design 5.29.3**, `@ant-design/icons` 5.6.1 | `apps/ui/package.json` |
| Fonts | `@fontsource-variable/inter`, `@fontsource/titillium-web` (bundled, CSP forbids remote) | `apps/ui/src/main.tsx:2-4` |
| Router | Own hash router (`#/jobs/...`), `useSyncExternalStore` on `hashchange`. No react-router | `apps/ui/src/app/router.ts` |
| State | Own read cache `useApi(key, fetcher)` + `invalidate(prefix)`; app-wide hooks in `session.ts`; feed filter in localStorage `jobleft.feed.v1`. No Redux/Zustand/React Query | `app/data.ts:66`, `:90`; `app/session.ts:10-23`, `:88` |
| Overlays | Esc stack + dirty-editor guard | `app/layers.ts` |
| Charts | **No chart library.** CSS div bars (`screens/Dashboard.tsx:67-87`), hand SVG ring (`components/Match.tsx:37`), antd `Progress` (`Dashboard.tsx:101`), SVG art/icons (`components/Art.tsx`, `Icons.tsx`) | |

### Folder layout (`apps/ui/`)
- `src/main.tsx` entry: `initToken()` `:22`, `<ConfigProvider theme={THEME}>` `:26`, antd `App` bridge → `layers.ts` `ui.message/ui.modal`.
- `src/app/` `App.tsx` (screen switch), `api.ts`, `data.ts`, `session.ts`, `router.ts`, `layers.ts`, `theme.ts`, `styles.css`.
- `src/components/` shared: `Shell.tsx` (Rail, TopBar, ConnectionBanner, ChatPanel), `JobCard.tsx`, `Match.tsx`, `VirtualList.tsx`, `States.tsx` (Loading/Empty/Error/Conflict/ErrorBoundary), `Chat.tsx` + `chatStore.ts`, `AiNote.tsx` (cost/consent note), `DraftModal.tsx`, `Tip.tsx`, `Icons.tsx`, `Art.tsx`.
- `src/screens/` one file or folder per screen; `src/lib/` pure helpers (unit-tested in `apps/ui/test/`).
- `src/index.ts` `SCREENS` (`:10`) = hash list; React-free so Node tests can import it.
- `mock/` full API mock + stand-ins (`pnpm --filter @jobleft/ui demo`); `scripts/` headless-Chrome probes/shots.

### Theme / tokens
- antd tokens + per-component overrides: `apps/ui/src/app/theme.ts:38` (`THEME`); raw palette `COLORS` `theme.ts:7`; fonts `:35-36`.
- The same palette is **duplicated** as CSS vars `--jl-*` in `apps/ui/src/app/styles.css:3-27`; all `jl-*` classes live in that one 312-line file. Change both.

### API access (`apps/ui/src/app/api.ts`)
- `initToken` `:19`: reads `#token=`, stores in memory + `sessionStorage['jobleft.token']`, rewrites hash to `#/jobs`.
- `client = createLocalApiClient({origin, launchToken})` `:36` (typed from `LOCAL_API`, `packages/contracts/src/client.ts:44`).
- `call(name, {params, query, body})` `:78` → typed response or throws `UiError` (plain sentence); `download(name)` `:90` for file routes; `streamChat` `:107` (SSE by hand).
- Pattern in screens: `const x = useApi('key', () => call('route'))` → render `x.data/x.error`; after a write `invalidate('key-prefix')`.

### Screens (route → file → main API routes → key components)
| Hash | File | Main routes (`call(...)`) | Key components |
|---|---|---|---|
| `#/jobs` (Recommended) | `screens/jobs/JobsScreen.tsx` → `Feed.tsx` | `searchJobs`, `crawlRun`, `storage`; side: `listFilters`, `createFilter`, `updateFilter`, `listResumes`, `listContacts`; `placeLookup` | `Filters.tsx` (FilterBar, SortControl, PlacePicker), `AllFilters.tsx`, `Side.tsx`, `cardActions.tsx`, `JobCard`, `VirtualList` |
| `#/jobs/liked\|applied\|external\|hidden` | `screens/jobs/TrackerTabs.tsx` | `listTracker`, `updateTracker`, `addExternalJob`, `getMatch` | `JobCard`, `useTrackerView` |
| `#/jobs/<jobId>` (overlay) | `screens/detail/JobDetail.tsx` | `getJob`, `updateTracker`; panels: `refreshCompany`, `listContacts`, `rankContacts`; tools: `tailorResume`, `acceptTailoring`, `keywordGaps`, `*CoverLetter` | `detail/Panels.tsx` (Company, Sponsor, Network, Notes), `detail/Tools.tsx` (Tailor/CoverLetter/Gaps drawers), `Match` |
| `#/tracker` | `screens/Tracker.tsx` | `listTracker` (via `useTrackerView`), `updateTracker` | `lib/trackerView.ts`, `lib/trackerEdit.ts` |
| `#/dashboard` | `screens/Dashboard.tsx` | `listTracker`, `searchJobs`, `storage`, `crawlReport`, `listContacts`, `listFilters` | `Kpi`, CSS bars |
| `#/resume`, `#/resume/<id>` | `screens/Resume.tsx`, `ResumeEditor.tsx` | `listResumes`, `importResume`, `createResume`, `updateResume`, `deleteResume`, `exportResume`, `atsCheck`, `fitCheck`, `listCoverLetters` | `AddResumeModal`, `resume/Extras.tsx`, `ReportDrawer` |
| `#/profile` | `screens/Profile.tsx` | `getProfile`, `putProfile` | `EditDrawer`, `lib/profileErrors.ts` |
| `#/onboarding` | `screens/Onboarding.tsx` | `putOnboarding`, `putProfile`, `importResume`, `getResume`, `updateResume` | `lib/onboarding.ts`; gate in `App.tsx:51-60` |
| `#/network[/people\|plan\|followups]` | `screens/Network.tsx` | `networkCoverage`, `networkCompanies`, `listContacts`, `rankContacts`, `networkPlan`, `planTopContacts`, `deleteContact`; `importNetwork`, `deleteNetwork` | `network/Import.tsx`, `network/shared.tsx`, `DraftModal` (`draftOutreach`, `previewDraft`) |
| `#/interview` | `screens/Interview.tsx` | `startPractice`, `practiceFeedback`, `*PracticeItem`, `listTracker`, `getJob` | `Question`, `Bank`, `AiNote` |
| `#/assistant[/<chatId>]` + panel everywhere | `screens/Assistant.tsx`, `components/Chat.tsx` | `listChats`, `getChat`, `deleteChat`, `chat` (SSE via `streamChat`), `decideProposal`, `cancelAi` | `ChatPanel` in `Shell.tsx` |
| `#/notifications` | `screens/Notifications.tsx` | `listNotifications`, `ackNotification` | |
| `#/settings/<tab>` (`#/boards` = sources) | `screens/Settings.tsx` (tabs `:31-37`: ai, balance, alerts, sources, data, extension, about) | AI: `putAiSettings`, `setAiKey`, `checkAi`, `*Publik`; sources: `listBoards`, `resolveBoard`, `addBoard`, `crawlRun`, `crawlReport`; data: `backup`, `restore`, `exportAll`, `deleteAllData`; ext: `pairingCode`, `listPairings`, `deletePairing` | `Panel`, one `*Tab` function per tab |

App-wide polls (`session.ts:46`): `crawlStatus` every 2-5 s, `listNotifications` every 12 s; reload-all on window focus.

### Add a screen end to end
1. **New data?** Add route to `LOCAL_API` (`packages/contracts/src/api.ts:197`) + schema in `packages/contracts/src/*.ts`; `pnpm --filter @jobleft/contracts gen` (writes `packages/contracts/schemas/*.json`). Additive only (INTERFACES §1.4).
2. Handler in `apps/server/src/routes.ts` `HANDLERS` (compile error until you do). Logic in a service on `AppData` (`app.ts:53`); new table = new entry in `SERVER_MIGRATIONS` (`db/schema.ts:24`, next version, never edit old steps).
3. Optional: mock handler in `apps/ui/mock/routes.ts:387` (`Partial`, so demo mode just 404s without it).
4. Screen `apps/ui/src/screens/Foo.tsx`: `useApi('foo', () => call('getFoo'))`, writes via `call(...)` then `invalidate('foo')`; use `States.tsx` for loading/empty/error.
5. Wire: `ScreenId` + `NAV` (`components/Shell.tsx:20`, `:22`) + icon (`Icons.tsx`), `case` in `App.tsx:72-98`, the head list at `App.tsx:99`, `WINDOW_TITLES` `App.tsx:37`, `SCREENS` in `src/index.ts:10`.
6. `node scripts/gen-interfaces.ts` (regenerates INTERFACES route table), `pnpm typecheck`, `pnpm --filter @jobleft/ui build`, reload.

## 3. Extension (`@jobleft/extension`, Chrome MV3; contract INTERFACES §7, README `apps/extension/README.md`)
- Pairing: Settings → Browser extension calls `pairingCode` (6 digits, 5 min, + port `routes.ts:374`); user types code + port in popup (`src/popup.ts`) → worker `pair()` `src/background.ts:78` → `POST /api/v1/extension/pair`; token only in `chrome.storage.local` (trusted contexts `background.ts:52`). All app traffic: `src/appclient.ts` (paired port only).
- Content scripts: none declared in `manifest.json`; `content.js` injected on click via `chrome.scripting.executeScript` (`background.ts:96`). Files: `src/content/main.ts` (collect/apply/messages), `fill.ts`, `dom.ts` (field discovery), `combobox.ts`, `panel.ts` (in-page report), `undoplan.ts`. Message router `background.ts:298`.
- Supported ATS (`src/support.ts:49`): Greenhouse, Lever, Ashby, Workable = supported; Workday, iCIMS = partial; LinkedIn/Indeed/Glassdoor never (`support.ts:33`). URL→ATS `support.ts:59`.
- App side: `answerFill` (`src/answer.ts`) is imported by the server (`services/extension.ts`); so the extension package is also a server dependency (packed into the sidecar).
- Build: `pnpm --filter @jobleft/extension build` → `apps/extension/dist` (esbuild, unminified), load unpacked.

## 4. Shell (`@jobleft/shell`, Tauri v2 + Node sidecar)
- Rust: `apps/shell/src-tauri/src/lib.rs`. `run()` `:388` → `spawn_server` `:158` (`node[.exe]` beside exe `:119` + `resources/server/src/main.js`, env `JOBLEFT_HOME`, token, parent pid, `JOBLEFT_QUIET=1`, UI dir; stdout/stderr → `logs/sidecar.log`) → `wait_ready` `:196` (run file + health, 15 s) → `open_window` `:226` (External URL `http://127.0.0.1:<port>/#token=...`; close = hide) → tray menu `:442-475` → 60 s poll of notifications `:490`. Quit: `stop_server` `:256` (`POST /shutdown`, SIGTERM on Unix, 5 s, kill).
- The webview is a plain browser page: `withGlobalTauri: false`, no Tauri commands; the UI never talks to Rust.
- Node side: `apps/shell/src/index.ts` (constants only). Sidecar tree: `apps/shell/scripts/pack.ts` (copies server + every `@jobleft/*` dep, transpiles `.ts`→`.js` with Vite's esbuild, flattens third-party deps, keeps only target's onnxruntime binary, copies `apps/ui/dist`, writes `resources/publik-app-token.txt`).
- Mac-specific: `osascript` notifications (`lib.rs:298`), dock Reopen (`lib.rs:515`), `Entitlements.plist`, `build-macos.sh` (sign + notarize), bundle targets `app`/`dmg` in `tauri.conf.json`, `app:build`/`tauri` package scripts (POSIX shell syntax).
- Windows-specific: toast via `tauri-plugin-notification` (`Cargo.toml:29-31`, `lib.rs:304`), `\\?\` path stripping (`lib.rs:144`), NSIS `currentUser` + WebView2 bootstrapper (`tauri.conf.json:42-51`).

### How the Windows package is produced (`.github/workflows/windows.yml`)
Runs on push to main, PRs, releases, manual. Steps:
1. `npm i -g pnpm@12.4.1` (`:42-48`), Rust stable + cache, `pnpm install --frozen-lockfile --ignore-scripts` (`:57`).
2. Every package's `node --test --test-timeout=180000 --test-force-exit "test/*.test.ts"` (`:64-82`, continue-on-error, verdict at the end `:161`).
3. `pnpm --filter @jobleft/ui build` → `node apps/shell/scripts/fetch-node.mjs --target win-x64` (Node v24.18.0, sha256 checked, → `src-tauri/binaries/node-x86_64-pc-windows-msvc.exe`) → `node apps/shell/scripts/pack.ts --target win-x64` (secret `PUBLIK_APP_TOKEN`).
4. `cd apps/shell; node <@tauri-apps/cli/tauri.js> build --bundles nsis` (**release**, default target dir) → `apps/shell/src-tauri/target/release/bundle/nsis/jobleft_<ver>_x64-setup.exe`.
5. Smoke `:111-141`: silent install → `%LOCALAPPDATA%\jobleft\jobleft.exe` (+ `node.exe`, `server\`, `ui\` beside it), start with `JOBLEFT_AUTO_CRAWL=0`, `JOBLEFT_SEED_BOARDS=none`, read `%APPDATA%\jobleft\run\server.json`, health, `POST /shutdown`.
6. Artifact `jobleft-windows-setup`; attached to the release on release events. Job `replay` `:169`: installs it again, runs `qa/bin/run-all.mjs` scenarios (live crawl of 12 boards).
Local repro on Windows: same commands (needs Rust MSVC toolchain + VS Build Tools; fetch-node downloads from nodejs.org). Unsigned installer.

## 5. Dev loop on Windows 11 (PowerShell, repo root)
| Want | Command |
|---|---|
| Install | `pnpm install --frozen-lockfile --ignore-scripts` (pnpm 12.4.1, Node >= 24; scripts are off by policy anyway) |
| Typecheck all | `pnpm typecheck` (TypeScript 7.0.2, `noEmit`) |
| Test all | `pnpm test` (= `pnpm -r --if-present test`); both: `pnpm check` |
| One package | `pnpm --filter @jobleft/server test` (`ui`, `extension`, `shell`, `contracts`, ...) |
| One file | `cd apps/server; node --test test/security.test.ts` (add `--test-name-pattern "<text>"`) |
| Run app in browser | `pnpm start` (= UI build + `node scripts/app-up.ts`); prints `http://127.0.0.1:<port>/#token=...` |
| Stop | `pnpm app:down` (see gotcha 6) |
| UI change | `pnpm --filter @jobleft/ui build`, then reload the tab (dist read per request, token survives in sessionStorage) |
| UI without real server | `pnpm --filter @jobleft/ui demo -- --persona` (mock API + stand-ins, data in `apps/ui/.mock-home`) |
| Seed a test profile | `node scripts/app-up.ts --persona backend-remote` (or `nurse-tx`) |

- Dev data: `<repo>/.jobleft-dev` (logs in `.jobleft-dev/logs/server.log`). Installed app: `%APPDATA%\jobleft`. Separate folders, both pick ports 47821-47830 (second one gets the next port).
- Env vars that matter (PowerShell `$env:X='1'; pnpm app:up`; full list INTERFACES §4): `JOBLEFT_HOME`, `JOBLEFT_PORT`, `JOBLEFT_OFFLINE=1` (no outbound at all), `JOBLEFT_AUTO_CRAWL=0` (no automatic live crawl), `JOBLEFT_SEED_BOARDS=none` / `JOBLEFT_SEED_LIMIT=<n>` (`core/seed.ts:88`, `:95`), `JOBLEFT_LOG_LEVEL=debug`, `JOBLEFT_UI_DIR`, `JOBLEFT_SECRET_STORE`, `JOBLEFT_CHROME` (browser drivers). `app:up` always sets `JOBLEFT_DEV=1`, `JOBLEFT_QUIET=1` (`scripts/app-up.ts:91`).

Mac-only / breaks on Windows:
- **AI keys are memory-only on Windows**: `createSecretStore` (`apps/server/src/services/secrets.ts:98-104`) is Keychain on macOS, memory elsewhere; `JOBLEFT_SECRET_STORE=file` is not honoured by the server (only `memory`/`keychain`). Keys vanish on every restart.
- `pnpm --filter @jobleft/shell app:build` / `tauri` scripts use `$(...)` and `VAR=x cmd` → fail under cmd/PowerShell. Use the windows.yml commands.
- Bash-only: `apps/shell/scripts/build-macos.sh`, `apps/server/scripts/jobsync-tests.sh`, `evals/ai-engine/hostile/run.sh` (+ macOS `sandbox-exec` profile `lib/loopback-only.sb`), `evals/gate8-shell/winshot.swift`.
- Hard-coded Mac paths: UI probes/shots Chrome path `apps/ui/scripts/browser.ts:8`; extension e2e `/private/tmp` (`apps/extension/scripts/e2e.ts:65`, `e2e-app.ts:31`, `cdp.ts:52`, `practice-server.ts:75`); `evals/gate*/run.mjs` use `/private/tmp`. `qa/bin/driver.mjs:4` does handle Windows.
- File modes 0700/0600 and `umask` are no-ops on Windows; `PRAGMA fullfsync` is macOS-only (harmless).

## 6. Tests, evals, qa
| Kind | Where | Run |
|---|---|---|
| Unit/integration | `apps/*/test/*.test.ts`, `packages/*/test` (`node:test`, scratch data folder under `os.tmpdir()` on Windows, `apps/server/test/helpers.ts:13`) | `pnpm test`, or per package as above |
| UI outcome probes | `apps/ui/scripts/probe/*.ts` (headless Chrome vs the mock; Mac Chrome path) | `node apps/ui/scripts/probe/run.ts --only nav,facts` |
| Extension e2e | `apps/extension/scripts/e2e.ts`, `e2e-app.ts` | `pnpm --filter @jobleft/extension e2e` (Mac paths) |
| Gates (build acceptance) | `evals/gate2-core` … `gate9-system`: `run.mjs` + `RESULT.md` (+ `gate7-ui/shots`, `gate8-shell/shots`) | `node evals/gate<N>-*/run.mjs`; Mac-bound (`/private/tmp`, gates 7-9 reuse gate 2's crawl) |
| Quality probes | `evals/match/ranking-pairs/run.ts`, `evals/network/csv-fixtures/run.ts`, `evals/filters-audit/run.mjs`, `evals/ai-engine/hostile/` (Python) | `node evals/<area>/<probe>/run.ts`; rules `evals/README.md` |
| Black-box QA | `qa/bin/*.mjs` (CDP drivers, prebuilt with esbuild), `qa/scenarios/*.mjs` per surface, `qa/practice/*.html` forms | set `JOBLEFT_QA_URL`, `JOBLEFT_QA_API`, `JOBLEFT_QA_TOKEN`; `node qa/bin/run-all.mjs [feed tracker ...]`; CI job `replay` |
CI: `.github/workflows/ci.yml` = typecheck + test on macos-latest and windows-latest; `windows.yml` as §4.

## 7. Gotchas
1. **Hash routes only.** The server has no SPA fallback (`static.ts:70-73`): any path-based route (`/jobs`) 404s on reload. A merged CareerOS UI must route by `#/...` or ship a real file per path.
2. **CSP blocks inline scripts and eval** (`static.ts:38`). Next.js static export injects inline bootstrap scripts, some chart libs use `eval`/`new Function`, remote fonts/CDNs are blocked by `connect-src`/`font-src 'self'`. Inline styles are fine (antd, most chart libs). Change the CSP in `static.ts` deliberately, never by accident.
3. **Browser code may import only `@jobleft/contracts`** (INTERFACES §1.1); the UI gets types + routes + client from it as TS source.
4. **Token lives in the fragment + per-tab sessionStorage.** Opening `http://127.0.0.1:<port>/` with no `#token=` shows the "Open from its launcher" screen (`App.tsx:25`). Every `pnpm app:up` restart makes a new token → old tabs get 401.
5. **Vite dev server (`pnpm --filter @jobleft/ui dev`, port 5174, proxy to :47821 `vite.config.ts:23-28`) is a different origin**; the Origin gate (`server.ts:210-217`, same in `mock/server.ts:186`) will likely refuse writes from it. Not verified; the proven loop is build + reload.
6. `pnpm app:down` sends SIGTERM (`scripts/app-down.ts:44`); on Windows that is a hard kill (no WAL checkpoint, lock left for stale takeover). Clean stop: `POST /api/v1/shutdown` with `x-jobleft-token` (the shell and CI do this).
7. **Fresh dev folder + first profile save = live crawl** of up to 120 real employer boards (`app.ts:168-183`). Set `JOBLEFT_AUTO_CRAWL=0` or `JOBLEFT_SEED_BOARDS=none` for UI work, `JOBLEFT_OFFLINE=1` for zero egress.
8. UI dir is picked at server start (`server.ts:132`): if `apps/ui/dist` did not exist then, you get the fallback page until restart.
9. **Newer schema = refusal** (`db/open.ts:97`, exit 2). Never point an older installed `jobleft.exe` at a folder your dev build migrated; keep dev on `.jobleft-dev`.
10. Palette lives twice (`theme.ts:7` and `styles.css:3`); layout classes are global `jl-*` in one CSS file, not CSS modules.
11. Most job/tracker/profile/boards logic is **INTERIM** (`apps/server/src/interim/`, `srv_*` tables) waiting to be replaced by the lane packages (`apps/server/README.md` §13); read the owner column in INTERFACES §6.4 before changing a route.
12. Pinned majors: React 18 + antd 5 + Vite 7 + TS 7. Mixing a React 19 codebase in means one of them moves.
13. `apps/server/jobsync/` looks like live Next.js code but nothing runs it; grep results from it are noise (exclude it).
