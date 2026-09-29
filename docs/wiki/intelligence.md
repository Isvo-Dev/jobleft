# Wiki: the intelligence layer

Covers `packages/{contracts,match,resume,ai-engine,assistant,network}`. This is a map for finding code. It does not repeat the specs, so read
these alongside it:
- `docs/INTERFACES.md`: §1 (runtime, dependency graph, additive-only rules, "null = unknown"), §4 (env vars), §8 package sections
  (contracts, ai-engine, resume, match, network)
- `docs/outcomes/<pkg>.md` and `i-ai.md`, `i-resume.md`, `i-network.md`: the acceptance rules (O-numbers)
- `docs/PLAN.md` §3 "Matching and AI": M1 score, M2 semantic, M3 copilot, M4 ATS, M5 network, M6 interview, T1 truth
- Package READMEs for match, resume, ai-engine and network. `assistant` has **no README and no INTERFACES section**, even though its code links
  to both.

Paths are relative to `packages/` unless they start with `apps/`, `docs/` or `scripts/`. A short path such as `score.ts:972` means the file in
the package named by the section.

## 1. contracts: schema → type → validation → routes

| What | Where |
|---|---|
| Builder (`str num int bool enm lit arr obj rec nullable union named`). Each call returns a JSON Schema carrying a phantom TS type | `contracts/src/schema.ts:81-143` |
| `Infer<typeof XSchema>` gives the TS type. `obj(required, optional)`: keys in the 2nd arg are optional, and **unknown keys are always allowed** | `schema.ts:42`, `:115` |
| `validate / isValid / parse` (throws `ContractError`) | `contracts/src/validate.ts:136-148` |
| Registry `SCHEMAS` → `schemas/*.schema.json` | `contracts/src/registry.ts:37`; writer `contracts/scripts/gen-schemas.ts`; staleness test `contracts/test/schemas.test.ts` |
| `LOCAL_API`: one `route({method,path,auth,owner,summary,query?,body?,response})` per route. `response` is a schema, `'sse'` or `'file'`. `body:{raw:[mime]}` means an upload | `contracts/src/api.ts:178` (RouteSpec), `:197-342` (table) |
| `RouteBody/RouteQuery/RouteResponse<K>`, plus `matchRoute` (literal segments win) and `buildPath` | `api.ts:344-393` |
| Typed client (UI, extension, tests, and the assistant's `DataApi`) | `contracts/src/client.ts`; UI: `apps/ui/src/app/api.ts` |
| `MATCH_BANDS` (85/70), `bandFor`, `summarizeMatch` | `contracts/src/match.ts:12`, `:183`, `:190` |
| `SECRET_NAMES` (`jobleft.ai.<slot>.key` …), `CONTRACTS_VERSION` | `contracts/src/ai.ts:230`, `contracts/src/index.ts:15` |

Generating docs:
1. `pnpm --filter @jobleft/contracts run gen` rewrites `schemas/`.
2. `pnpm gen:interfaces` (`scripts/gen-interfaces.ts`) rewrites the INTERFACES.md blocks between `<!-- BEGIN GENERATED: routes -->` and
   `sig:<pkg dir>`. The routes table comes from `LOCAL_API`. Signatures come from `tsc --emitDeclarationOnly` of each `src/index.ts`. Text
   outside these blocks is written by hand and kept as it is.
3. No `sig:` block exists for contracts or assistant. To get one, add a marker pair to INTERFACES.md.

## 2. match: the deterministic score (M1)

Entry: `scoreMatch` → `computeMatch` at `match/src/score.ts:972`. It is pure: no network, no AI. Profile facts are cached per month (`factsFor`
`:954`).

| Step | Function | File:line |
|---|---|---|
| Profile → facts (roles, months, held skills, degrees; never EEO or contact data) | `profileFacts` | `match/src/profile.ts:262` |
| `profileVersion` hash of only the fields the score reads | `scoringView` / `profileVersionOf` | `profile.ts:102`, `:121` |
| Job → facts (sections, skills with importance, level from title, family, industry, language) | `readJob` | `match/src/job.ts:324` |
| Text analysis. Sentences aimed at AI screeners are dropped here | `analyzeText`, `isInjection` | `match/src/text.ts:237`, `:168` |
| Posted requirements (years, degree, licence, "or equivalent") | `readRequirements` | `match/src/requirements.ts:210` |
| Taxonomy (TSVs in `match/data/`) | `scanSkills`, `familyOfTitle`, `scanIndustries` | `match/src/taxonomy.ts:386`, `:464`, `:542` |
| **Experience Level** (does the kind of work match × does the level match) | `scoreExperience` | `score.ts:230` |
| **Skills** (weighted coverage: required 1 / preferred .5 / mentioned .6 / related half credit) | `scoreSkills` | `score.ts:436` |
| **Industry Experience** | `scoreIndustry` | `score.ts:546` |
| Must-haves → caps and blockers; deal-breakers (work model, place, pay, type); chips | `evaluateMustHaves`, `evaluateDealBreakers`, `chips` | `score.ts:595`, `:786`, `:882` |

Formula (`score.ts:980-1016`):
1. `raw = intercept + wE·E + wS·S + wI·I`. A `null` part adds 0, goes into `unknownParts`, and sets `complete=false`, so an incomplete score is
   a floor.
2. Add the optional semantic term.
3. Apply the **lowest** applicable cap.
4. `band = bandFor(percent)`.

Every step pushes a `Reason` (`formula`, `semantic`, `cap`, `incomplete`), so the detail view shows the arithmetic. Full rules: `match/README.md` §4.
- Numbers: `DEFAULT_CONFIG` `match/src/config.ts:64`. Weights E .24, S .29, I .08, +36, semantic 0. Caps: legal 45, licence 70,
  degree/years 65, level/deal-breaker 60, not-in-profile 84, step-down 72. A non-default config re-tags the version as
  `match-1.0.0+cfg-<hash>` (`configTag` `config.ts:89`).
- **Fit model (M2)**: vectors are read only if `weights.semantic > 0` (`score.ts:994-1001`, cosine mapped over 0.2 to 0.8). It is off on
  purpose: a vector that arrives late would change a score with no data change (README §6). Embedder: `createLocalEmbedder` (bge-small-en-v1.5,
  ONNX loaded at run time, `not_ready` without it) `ai-engine/src/embedder.ts:183`. Index and Top Matched cosine sort: `store/src/fit.ts`,
  `store/src/vectors.ts`, `store/src/embed/` (outside this scope).
- Extras: `rankTopMatched`/`bucketOf`/`bandCounts` (`match/src/rank.ts`), `setSkillClaim`/`undoSkillClaim` (`match/src/claims.ts:27`). The only
  allowed AI-summary path: `narrativeBrief`/`checkNarrative` (`match/src/narrative.ts:25`, `:71`).
- Server call sites: `apps/server/src/integ/match.ts:10`, `apps/server/src/core/feed.ts:438`.

## 3. resume: import → profile → tailor (truth gate) → export → ATS (M4, T1)

| Stage | Code | Notes |
|---|---|---|
| Import (worker thread, 30 s, 10 MB) | `resume/src/import/index.ts:151` → `importInProcess` `:77` → `import/worker.ts` | File type sniffed from its bytes, not its name |
| PDF read | `resume/src/pdf-read.ts:48` (unpdf/PDF.js, offline) → `import/lines.ts:143` lines, `:104` columns, `:206` page furniture | Max 12 pages |
| DOCX read | `import/lines.ts:357` over `resume/src/zip.ts:83` (own zip reader with limits) | `.doc`/`.rtf`/images refused with a plain message |
| Lines → `ProfileInput` + `ImportReport` | `import/parse.ts:472` `parseLines`; `import/dates.ts`; section names in `resume/src/lexicon.ts` | |
| Save upload + **proposed** profile | `ResumeService.import` `resume/src/service.ts:162` | Never writes the profile. The caller saves it only after the person confirms |
| Base doc from profile | `resume/src/document.ts:19` | Header copied from the profile exactly |
| Keyword gaps | `resume/src/gaps.ts:121` (`builtinSkillDictionary` `:14`) | |
| Tailor draft | `draftTailoring` `resume/src/tailor.ts:192`; `TailorOp` `:20`; `applyChanges` `:347` | No AI: add/reorder skills and bullets. AI: reword bullets and summary, one call, no retry (`resume/src/ai.ts:30`) |
| Accept | `service.ts:308` | Saves a new `tailored` row; the base never changes |
| Cover letter | `draftLetter` `resume/src/letter.ts:205`, `editLetter` `:262` | The gate runs in letter mode |
| PDF | `render/index.ts:37` → `fitResume` `render/layout.ts:326` (`PASSES` `:20`, then whole items left out) → `writePdf` `render/pdf-writer.ts:124` | Always 1 page, left-out items listed, unprintable characters refused (TTF: `JOBLEFT_PDF_FONT`, `render/truetype.ts`) |
| DOCX | `render/index.ts:55` → `render/docx.ts:139` | Same fit as the PDF, so the same items |
| ATS | `atsCheckPdf` `resume/src/ats.ts:59` | Re-parses **the exported PDF bytes**. Parts: text 35, layout 20, sections 20, contact 15, dates 10 |

**Truth gate**: `resume/src/truth.ts`.
- Functions: `buildProfileFacts` `:106` builds the allowed corpus; `checkText` `:571`; `checkDocument` `:727`; `checkLetter` `:785`;
  `truthGate` `:825`; `refusedFacts(instruction)` `:835`.
- What it enforces: every skill, cert, degree, number, date, duration (≤ years the profile dates show), title, employer, school, place,
  contact detail and name must trace to the profile. The posting is **never** a source of facts. Job names may appear in a letter only when it
  talks about the job. The header must equal the profile.
- Where it runs:
  - Each AI change: `tailor.ts:304-318`, plus `numbersKept`, so a reworded bullet cannot add a number. Failures are dropped and listed in
    `violations`.
  - Again on accept, for facts the accepted changes *add*: `service.ts:326-336`.
  - Letters: `service.ts:459`, `:494`.
- Not gated: the person's own edits and uploads (`update()`).

## 4. ai-engine: providers, keys, structured output

Kinds (`contracts/src/ai.ts:12-14`):
- Providers: `publik`, `local`, `custom`, `own_key`.
- Local servers: `ollama`, `llamacpp`, `mlx`, `lmstudio`, `openai_compatible`.
- Vendors: `openai`, `anthropic`, `openrouter`, `google`.

| Choice | Driver (`driverFor` `ai-engine/src/engine.ts:305`) |
|---|---|
| publik (gateway key from `PublikClient`; cost from `x-publik-charge-micros`) | `OpenAiDriver` `providers/openai.ts:60` |
| local ollama | `OllamaDriver` `providers/ollama.ts:70` (native `/api/chat`, think option) |
| other local / custom URL | `OpenAiDriver` + `resolveOpenAiRoot` (probes `/v1`, caches the root) |
| own_key anthropic | `AnthropicDriver` `providers/anthropic.ts:44` |
| own_key openai / openrouter / google | `OpenAiDriver` on `VENDOR_BASE_URLS` `urls.ts:10` |

| Topic | Where / rule |
|---|---|
| Layers | `ProviderDriver` (`types.ts:86`: `stream`, `listModels`, `embed`) → `makeClient` (`client.ts:23`) → `AiClient` (`chat`, `complete`, `json`, `listModels`, `embed`), plus cancel-by-requestId, a publik guard and a balance re-read hook |
| Get a client | `engine.client()` `engine.ts:359`: the chosen provider or `no_provider` / `offline` / `key_refused` / `model_not_found`. **Never a fallback to another provider** |
| Settings | `updateSettings` `engine.ts:145` (local must be loopback, URLs with credentials refused), then `check()` `:421`. Stored by `kvSettingsStore` `state.ts:76`: server `srv_kv` `ai:*` (`apps/server/src/integ/engine.ts:42-52`), CLI `<home>/ai/state.json` (`setup.ts:27`) |
| Key slots | `keySlot` `engine.ts:118`: `own_key.<vendor>`, or `custom@<hash>` / `local@<hash>` of the origin. A new URL means the key is saved again |
| **Secrets by OS** | `osSecretStore` `secrets.ts:188`: macOS Keychain via `/usr/bin/security`, secret on stdin (`:60`). **Every other OS, Windows too:** `encryptedFileSecretStore`, AES-256-GCM `<home>/secrets/secrets.enc` + `master.key` (`:100`). Credential Manager is not wired (`:186`). `JOBLEFT_SECRET_STORE=keychain\|file\|memory` overrides |
| **Server secrets** | The server ignores the above. Its interim store (`apps/server/src/services/secrets.ts:98`) is **memory-only off macOS**, and a Keychain failure also falls back to memory. **Keys saved in the Windows app are lost at restart** |
| Structured output | `client.json({messages, schema, lineFallback})` `client.ts:68`: `jsonInstruction` `structured.ts:15`, then `readStructured` `:112` (lenient extract, `validate`, `lineFallback`). Nothing fits: `AiError('bad_answer')`, never a default. Also `parseScoresHeader` `:158`, `readFieldLines` `:181`, `stripThinking` (`thinking.ts`) |
| Errors | `toApiError` `errors.ts:58`: 409 needs_provider, 402 balance/claim with one link, 504 timeout, 502 provider_error |
| Mocks | `mock/model-server.ts:60` (OpenAI, Ollama and Anthropic dialects, 16 `MODEL_MODES`: stall, half, html, refuse-key, cutoff…); `mock/publik-server.ts:46`; `assistant/src/mock/scripted-model.ts` (tool calls; used by `assistant/test/helpers.ts`). `JOBLEFT_AI_HOST_MAP` (`urls.ts:90`, loopback only). CLI: `node packages/ai-engine/src/cli.ts` |

**Add a provider**
- Same wire format as OpenAI (a new vendor): `OwnKeyVendorSchema` (`contracts/src/ai.ts:14`), `VENDOR_BASE_URLS` (`urls.ts:10`),
  `VENDOR_LABEL` (`engine.ts:63`), the `own_key` case of `driverFor` (`engine.ts:330`), and the host in INTERFACES §9.
- New wire format: `providers/<x>.ts` implementing `ProviderDriver` (copy `anthropic.ts`; use `transport.ts` and `classify.ts`), a dialect in
  `mock/model-server.ts`, and cases in `ai-engine/test/providers.test.ts`.
- New top-level kind: additive contracts enum, plus `updateSettings`, `driverFor`, `keySlot` and `describe`.

**Change the default model**
- publik: `PUBLIK_DEFAULT_MODEL` / `PUBLIK_TIERS` (`publik.ts:20-21`).
- local: `LOCAL_DEFAULT_URLS` (`urls.ts:18`); the initial kind is ollama (`engine.ts:157`).
- own_key, custom and local have **no default model**: it stays null and `client()` throws `model_not_found`. To add one, set it in the
  `updateSettings` cases (`engine.ts:156-185`). Empty settings: `defaultAiSettings` `state.ts:61`.

## 5. assistant: chat with tools (M3, M6; outcomes i-ai)

- Wiring: `buildAssistant` (`apps/server/src/integ/assistant.ts:54`) creates `new Assistant({engine, api: inProcessApi(...), db})`. It reads and
  writes **only via LOCAL_API**: `DataApi.call` (`assistant/src/data.ts`) runs server `HANDLERS` in-process with contract checks
  (`integ/assistant.ts:16`). Tables `ai_chats`, `ai_chat_messages`, `practice_*` and `ai_usage` come from `assistant/src/db.ts` (owner `ai-engine`).
- Loop, `runChat` (`assistant/src/assistant.ts:141`): save the user message → `systemPrompt` (`prompts.ts:30`; rules `:7`; 7 `PRESETS` `:4`),
  plus `get_job` facts when `jobId` is set → stream `client.chat({tools})` at temperature 0.2 → run the tools → repeat. Limits:
  `MAX_ROUNDS=6`, `MAX_TOOL_CALLS=14`, 6 per round, 16 history messages (`:28-31`).

| Tool (`assistant/src/tools.ts`) | Where | Offered when |
|---|---|---|
| `search_jobs list_tracker get_job get_match get_profile company_facts list_contacts next_steps list_resumes tailor_resume interview_prep read_page` | defs `:63-76`, impls `:172-345` | Per-preset whitelist `toolsFor` `:100`. `read_page` is free (`fetchfree.ts` + `urlsafe.ts`) |
| `propose_changes` (the only write tool) | `:361` | The **user's own message** matches `CHANGE_INTENT` `:51` |
| `paid_lookup` (page/search) | `:384` | `WEB_INTENT` `:52` matches **and** `meteredFetch.enabled` |

Dispatch: `toolbox.run` `:415`. `fitResult` `:121` caps results at 12k chars. Posting and page text is wrapped under `untrusted`.

| Topic | Where / rule |
|---|---|
| Proposals | `ProposalBook` `proposals.ts:28`: memory only, 1 h TTL, so closing the app declines them. `buildAction` turns a model change into one checked `StoredAction`; the summary comes from records, not model text. Kinds: `CHANGE_KINDS` `:69`. `decideProposal` `assistant.ts:291` → `apply` `:314` (one LOCAL_API call per kind) → `decisionText` `:68` is appended to the chat |
| Grounding | `Grounder` `grounding.ts:7` filters the **stream**: numbers, $, %, links and dates not in this turn's tool results or the user's messages become a marker. "Does not sponsor" only if a source says it. `jobsNamedIn` `:118` makes job chips |
| Model-safe profile | `profileForModel` (`profileview.ts`): no name, contacts, work authorization or EEO data |
| Money | `UsageLedger` (`usage.ts`) records only charges publik reported. `createMeteredClient` (`metered.ts`): off by default, publik-gateway only, price stated first, never retries. Server fallback when the charge is unknown: balance before minus after (`AiFacade.metered` `apps/server/src/integ/engine.ts:62`, `assistantRoute` `integ/assistant.ts:70-85`) |
| Practice | `practice.ts`: questions come from **rules, not AI** (`planQuestions` `:93`, only skills the posting names, `PRACTICE_LABEL`). `feedbackFor` `:376`: one JSON-schema AI call, then `verifySample` (strips facts not in the answer or profile) and `stripEmployerClaims`. `no_provider` or `bad_answer` falls back to `rulesFeedback`. The answer is saved before the model call |
| Test stand-ins | `standin/world.ts` (in-memory LOCAL_API), `mock/scripted-model.ts` |

## 6. network: CSV → ranking → drafts (M5)

- Parse: `parseConnectionsCsv` `network/src/csv.ts:198`. Handles BOM and encoding (`text.ts:12`), note lines and garbled names
  (`text.ts:83`). Every skipped row comes with a reason.
- Import: `NetworkService.import` `network/src/service.ts:219`. Identity is the profile URL, else name plus date. Stages and notes survive a
  re-import.
- Company keys: `network/src/company.ts`: `networkKey` `:145`, `keysForCompany` `:194`, placeholders `:177`, `howMatched` / `whyNotCounted`.
  It uses static-data's `companyKey`, with an interim fallback `resolveCompanyKey` `:63`.
- Rank: `scoreContact` / `rankContacts` (`network/src/rank.ts:50`, `:127`) with `RANK_POINTS` `:11`; titles via `readTitle`
  (`network/src/titles.ts:128`). Each reason quotes the file.
- Drafts (`network/src/draft.ts`):
  1. `draftFacts` `:67` passes only the contact's name, title and company, the job, and `profileSummary` `:35`.
  2. `draftMessages` `:86`.
  3. `ai.complete` in `draftOutreach` `:419`.
  4. `cleanDraftText`, then `checkDraft` `:256` (invented facts, links, emails, 300/1200 chars). Any warning sets `ready=false`.
  - `templateDraft` `:390` needs no AI. A remote provider needs confirmation once per destination (`network/src/routes.ts:169-190`).
- Routes: `handleNetworkRoute` (`routes.ts:108`), called from `apps/server/src/routes.ts:113`.

## 7. How do I…

**Add an assistant tool (read-only)**
1. Add an `AiTool` to `READ_TOOLS` (`tools.ts:63`). Parameters are plain JSON Schema via the local `obj()`.
2. Add the implementation to `impl` in `createToolbox` (`tools.ts:171`). Read only via `data.api.call('<route>')`, call `seen()` for jobs,
   wrap free text as `untrusted`, and use null or a note for unknowns.
3. Add the tool to the `byPreset` lists (`tools.ts:101`). `chat` and `fit` get every read tool.
4. Add a stand-in answer to `standin/world.ts` and a test in `assistant/test/chat.test.ts` with the scripted model.
5. Writes are never new tools. Add a kind to `CHANGE_KINDS` (`proposals.ts:69`), a branch in `buildAction`, and a `case` in `Assistant.apply`
   (`assistant.ts:317`). Extend `ActionProposalSchema` kinds (`contracts/src/ai.ts:100`), additive only.

**Add an AI feature end to end (for example a coaching endpoint)**
1. Contract:
   - Add schemas in `contracts/src/ai.ts` (or a new file re-exported by `contracts/src/index.ts`) and register them in `SCHEMAS`.
   - Add a `route({...})` in `LOCAL_API`. `owner` must be a `RouteOwner` (`api.ts:174`). There is no `'assistant'` owner; assistant routes use
     `'ai-engine'`.
   - Bump `CONTRACTS_VERSION`, then run `gen` and `gen:interfaces`.
2. Logic: add a method on `Assistant` (it has `engine`, `data`, `usage`). Copy the pattern of `feedbackFor` (`practice.ts:376`):
   - pass the profile only via `profileForModel`, and fence data blocks;
   - use `withJsonInstruction` + `readStructured`, or `client.json`;
   - post-check the output (`truthGate` from `@jobleft/resume`, `checkNarrative` from `@jobleft/match`);
   - handle `no_provider` with a rules fallback, and call `usage.record(cost)`.
3. Handler: add the name to `ASSISTANT_ROUTES` and `createAssistantHandlers` (`assistant/src/routes.ts:17`, `:47`). The map is typed, so tsc
   fails until it is filled. Validate with `checkBody`.
4. Server: add `(c) => assistantRoute(c, '<name>')` to `HANDLERS` (`apps/server/src/routes.ts:145`). `HandlerTable` is keyed by
   `RouteName`, so tsc forces the entry. For a non-streamed publik cost, wrap the call in `d.ai.metered(...)` as `tailorResume` does
   (`routes.ts:276`).
5. UI: call it through `apps/ui/src/app/api.ts`. Tests use `startMockModelServer` or the scripted model, never a real vendor.
6. A new package edge (for example assistant → resume) is not in INTERFACES §1.2, and assistant itself is missing from that table. Agree the
   edge before adding it.

**Change match weighting**
- Edit `DEFAULT_CONFIG` (`match/src/config.ts:64`), or pass `config` per call. Weight changes re-tag `engineVersion` by themselves.
- Bump `ENGINE_BASE_VERSION` (`match/src/score.ts:18`) when you change rules.
- Re-run `node --test packages/match/test/*.test.ts` and the probe `evals/match/ranking-pairs`.
- To enable semantic, set `weights.semantic > 0` **and** pass vectors. No server call site passes them today.

**Add a resume export template**
There is no template system: one layout lives in `render/layout.ts` (`layoutResume` `:232`, `PASSES` `:20`) plus `render/docx.ts:139`.
1. Add an optional `template` query field to `exportResume` (`contracts/src/api.ts:250`, additive).
2. Thread it through `ResumeService.export` (`service.ts:379`) and the server handler.
3. Add a layout function or `LayoutParams` set.
4. Keep DOCX parity: both formats go through `fitResume`.
5. Keep one page, never cut text, and list items left out.
6. `atsCheck` (`service.ts:400`) grades `export(id,'pdf')`, so decide which template it grades.
7. Add a case to `resume/test/render.test.ts` and confirm the ATS parser still reads the layout.

## 8. Gotchas

- **Additive-only contracts** (INTERFACES §1.4). `obj()` accepts unknown keys, so a misspelled optional field passes validation silently.
- **The chat tailoring instruction is dropped.** The `tailor_resume` tool sends `{jobId, instruction}` (`tools.ts:320`), but the route body is
  `{jobId}` (`contracts/src/api.ts:247`) and the server passes only `body.jobId` (`apps/server/src/routes.ts:276`). `refusedFacts` never runs from chat.
- **Keys don't persist in the Windows app** (§4). The server uses memory; only the ai-engine CLI uses the encrypted file.
- **The server scores matches with `company: null`** (`integ/match.ts:10`, `core/feed.ts:438`). No company-data industry and no "H-1B sponsor
  likely" chip come through this path.
- `narrativeBrief` / `checkNarrative` exist, but no server or assistant code calls them. A new AI match summary must go through them.
- Permission to write or use the web comes from **regexes over the last user message** (`CHANGE_INTENT`, `WEB_INTENT`). Wording without one of
  the listed verbs gets no write tool.
- Grounding covers streamed chat only. Every other AI feature must post-check its own output.
- The truth gate checks AI drafts only. Uploaded or hand-edited resumes can hold facts that are not in the profile, and tailoring keeps those
  lines untouched.
- Proposals, cached OpenAI roots and the grounding corpus are per-process or per-turn. Proposals are lost on restart by design.
- `RANK_POINTS` has `sameRole: 15` (`network/src/rank.ts:16`), which INTERFACES §8 does not list. The code wins.
- `scripts/gen-interfaces.ts` runs `node_modules/.bin/tsc` via `execFileSync`. On Windows the extensionless shim probably fails; use Git Bash
  or point it at `tsc.cmd`.
- The assistant docs are stale: the code points to a README and an INTERFACES section that don't exist; `package.json` `bin: src/cli.ts` and
  `routes.ts`'s `dev/serve.ts` are also missing.
- The free page reader has its own UA (`assistant/src/fetchfree.ts:15`), separate from the crawler's fixed `USER_AGENT` rule (INTERFACES §4).
- Resume and network AI steps are one call with no retry, so a paid step is charged once. A failure saves nothing. Import, export and ATS never
  call a model.
