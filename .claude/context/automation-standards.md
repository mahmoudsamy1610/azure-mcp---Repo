# Automation Standards & Conventions

> The contract for the test-automation layer. Every automation agent and skill reads
> this before writing a line of code. It defines the framework structure, naming, the
> locator strategy, wrapper/Page-Object rules, reporting, and how automation maps back
> to the QA test cases. Project/business facts and QA tagging/IDs come from the
> project context — ask the user if not provided. This file governs **how
> the automation is built**, not what to test.

---

## Where the framework lives — read this first

The generated framework is **NOT part of this repository.** This git repo is the
MCP / QA-orchestration system only. The runnable automation framework is a **local,
generated artifact**:

- It is scaffolded at the **project root** (`./automation/`) **only when the user asks**
  (via the `scaffold-automation-framework` skill).
- It is **git-ignored** here — `automation/`, `allure-results/`, `allure-report/`,
  `.env`, videos, and screenshots never get committed to this repo.
- What *is* committed here is the **intelligence layer**: these standards, the two
  automation agents, and the automation skills. They know how to *produce* the
  framework; they are not the framework.

If `./automation/` does not exist yet, the correct move is to run
`scaffold-automation-framework`, not to hand-write files ad hoc.

---

## Stack — pinned

| Surface | Driver | Language | Runner | Report |
|---|---|---|---|---|
| **Web** (WOQOD / FAHES / Qjet sites, CMS) | **Playwright** | Python 3.11+ | **pytest** | **Allure** + Playwright trace/video |
| **Mobile** (app — iOS + Android) | **Appium** | Python 3.11+ | **pytest** | **Allure** + screen recording |

- One runner everywhere: **pytest**. One report everywhere: **Allure**.
- Surface is chosen **per feature**, per the project's service ↔ platform mapping —
  ask the user when a feature's surface is unclear. A feature on the app →
  mobile/Appium; a feature on a website or CMS → web/Playwright. A feature on both →
  a test in each tree, shared step intent.
- **API accuracy — docs over memory.** When unsure about a Playwright, Appium, pytest,
  or Allure API (signature, option name, deprecation), consult the **official
  documentation** (playwright.dev/python, appium.io, docs.pytest.org,
  allurereport.org) via WebFetch/WebSearch rather than coding from training memory.
  *(An offline docs snapshot as a local RAG source is under consideration; until then,
  the online docs are the reference.)*

---

## Repository structure (generated at `./automation/`)

```
automation/
├─ README.md                  # setup + run instructions (incl. mobile prereqs)
├─ requirements.txt           # pinned deps
├─ pytest.ini                 # markers, addopts (--alluredir, etc.)
├─ .env.example               # URLs, Appium caps, creds — copied to .env locally
├─ conftest.py                # root fixtures: config, driver/browser, hooks
├─ config/
│  └─ settings.py             # typed config loaded from env; NO secrets in code
├─ core/                      # the reusable framework — never feature-specific
│  ├─ web/
│  │  ├─ browser.py           # Playwright launch/context/page factory
│  │  └─ base_page.py         # BasePage wrapper (the only place raw Playwright is touched)
│  ├─ mobile/
│  │  ├─ driver.py            # Appium driver factory (iOS/Android caps)
│  │  └─ base_screen.py       # BaseScreen wrapper (the only place raw Appium is touched)
│  └─ utils/
│     ├─ reporting.py         # screenshot/video attach → Allure
│     ├─ waits.py             # explicit-wait helpers
│     └─ logger.py
├─ web/
│  ├─ pages/                  # Page Objects — grouped by page/module, never flat
│  │  ├─ components/          #   shared cross-page component objects (header, nav, OTP modal)
│  │  └─ login/               #   e.g. pages/login/login_page.py
│  └─ tests/                  # pytest tests — mirrors pages/, one folder per page/module
│     └─ login/               #   e.g. tests/login/test_login.py — ALL login cases in ONE module
├─ mobile/
│  ├─ screens/                # Screen Objects — grouped by screen/module, never flat
│  │  ├─ components/          #   shared cross-screen component objects
│  │  └─ tag_topup/           #   e.g. screens/tag_topup/tag_topup_screen.py
│  └─ tests/                  # mirrors screens/
│     └─ tag_topup/           #   e.g. tests/tag_topup/test_tag_topup.py — ALL top-up cases in ONE module
└─ reports/                   # generated allure-results / allure-report (git-ignored)
```

Rules:
- **`core/` is generic.** Nothing in `core/` references WOQOD, a specific page, or a
  specific feature. Feature knowledge lives only in `pages/`, `screens/`, and `tests/`.
- **Group by page/module — never flat.** Every Page/Screen Object and every test module
  lives in a folder named after the page/module it belongs to (`pages/login/`,
  `tests/login/`), with matching folder names between the object tree and the test tree
  (each folder gets an `__init__.py`). Folder names are snake_case; a service-scoped
  feature uses `<service>_<feature>` (e.g. `tag_topup`, `fahes_booking`). The folder is
  created with the first artifact for that page; new artifacts for an already-covered
  page go **into the existing folder**. The one exception to page-naming: reusable
  cross-page component objects (header, nav, OTP modal) live in `pages/components/` /
  `screens/components/` — never flat and never duplicated into page folders.
- **One spec module per page/feature — NEVER one file per test case.** The single test
  module's name mirrors its folder: `tests/<page>/test_<page>.py` (e.g.
  `tests/login/test_login.py`, `tests/tag_topup/test_tag_topup.py`). All test cases for
  a page/feature accumulate in that one module. The existing module is identified **by
  the page folder**: if any `test_*.py` already exists in `tests/<page>/`, append to it
  regardless of its exact name — creating a second module, or a new file per case, is a
  defect.
- **Tests never touch raw Playwright/Appium.** A test calls Page/Screen-Object methods,
  which call `BasePage`/`BaseScreen` wrappers. If a test imports `playwright` or
  `appium` directly, that's a defect.
- **No locators in tests.** Locators live only inside Page/Screen Objects.

---

## The wrapper layer (`BasePage` / `BaseScreen`)

Every interaction goes through a wrapper that is **self-waiting, self-logging, and
self-screenshotting**. The wrapper is where flakiness is killed and where Allure
attachments are produced.

Minimum wrapper API (both web and mobile mirror these):

| Method | Guarantees |
|---|---|
| `open(target)` / `launch()` | navigate / start session |
| `click(locator)` | wait-for-actionable → click → log |
| `type(locator, text)` | wait → clear → type → log (mask secrets) |
| `text(locator)` | wait-for-visible → return text |
| `is_visible(locator)` | bounded wait → bool, never throws |
| `wait_for(locator, state)` | explicit wait, no `sleep()` |
| `screenshot(name)` | capture → attach to Allure |
| `assert_visible / assert_text` | soft-fail aware, attaches on failure |

Hard rules:
- **No `time.sleep()`.** Ever. Use explicit waits from `core/utils/waits.py`.
- **No bare asserts in Page Objects.** Page Objects expose state; tests assert. (A small
  set of `assert_*` wrappers is fine for readability, but the assertion intent lives in
  the test.)
- **Every action logs** what it did and on which locator.
- **Secrets are masked** in logs and reports (passwords, OTP, card numbers, tokens).

---

## Locator strategy — priority order

Locators are **extracted on demand** by the `extract-locators` skill (never hand-guessed
in bulk, never committed as static placeholder dumps).

**Fetch via MCP — required mechanism.** Locator discovery drives the live app through
the MCP servers: the **Playwright MCP** for web (DOM + accessibility tree) and the
**Appium/mobile MCP** for the app (UI hierarchy). MCP inspection is far more
token/cost-efficient than screenshot-driven discovery or ad-hoc throwaway scripts —
prefer the text-based tree/find tools over screenshots. Fall back to a scripted
inspection only when the relevant MCP server is unavailable, and say so explicitly.

When choosing a locator, prefer the highest available tier:

**Web (Playwright):**
1. `data-testid` / `data-test` (ask the team to add these where missing — most stable)
2. Role + accessible name (`get_by_role`, `get_by_label`) — survives restyling
3. Stable `id`
4. Scoped CSS (short, semantic — no deep descendant chains)
5. **XPath — last resort only**, and only relative/text-anchored, never absolute

**Mobile (Appium):**
1. `accessibility id` (the cross-platform first choice)
2. Android: `resource-id`; iOS: `name` / predicate on stable attributes
3. `-android uiautomator` / iOS class chain for lists/repeating items
4. **XPath — last resort only** (slowest on mobile; flag it in the PO)

Locator hygiene:
- One named constant per element inside its Page/Screen Object — never inline a raw
  selector at a call site.
- Name locators by intent (`login_button`), not by tag (`blue_div`).
- RTL/Arabic: locate by `testid`/role/`accessibility id`, **never by visible Arabic
  text** unless the test is specifically asserting that text.

---

## Page Object / Screen Object rules

- **One class per page/screen/component.** File name = snake_case; class = PascalCase
  (`login_page.py` → `LoginPage`).
- A Page Object holds: its locators (constants) + action methods (`login(user, pwd)`) +
  state queries (`error_message_text()`). It returns the next Page Object on navigation.
- **No assertions, no test data, no waits-by-sleep** inside Page Objects.
- Reusable cross-feature components (header, nav, OTP modal) get their own component
  object in `pages/components/` / `screens/components/` and are composed in, not
  copy-pasted.

---

## Test structure & naming

- One test module per page/feature, inside that page's folder, module name mirroring
  the folder name: `tests/<page>/test_<page>.py`
  (e.g. `tests/tag_topup/test_tag_topup.py`, `tests/fahes_booking/test_fahes_booking.py`,
  `tests/login/test_login.py`).
- **All test cases for a page/feature live in that one module.** A new case for an
  already-covered page is **appended** to the existing module as a new test function —
  never a new file. Identify the module by the folder: append to whatever `test_*.py`
  already exists in `tests/<page>/`. One-file-per-test-case is a defect.
- One test function per scenario, named for intent:
  `test_topup_with_expired_card_is_rejected`.
- **AAA shape:** Arrange (fixtures/preconditions) → Act (Page-Object calls) →
  Assert (in the test).
- **Independent & idempotent:** no test depends on another's side effects; each sets up
  and tears down its own data. Parallel-safe.
- **Concrete data, mirrored from the QA case** — use the same concrete values the test
  case specifies (`Top-up = 50 QAR`), via a data builder/fixture, not hard-coded literals
  scattered in the test body.
- Every test carries the QA **traceability ID** in a marker/docstring
  (`# TAG-TOPUP-TC-014`) so an automated test maps back to its source case, **and the
  parent backlog item ID** as an `@pytest.mark.pbi_<id>` marker (Axis B below) so it
  also maps back to the PBI the case was derived from.

### pytest markers ↔ QA tag axes — every axis gets a mark

A test's markers are **derived mechanically from its source case's `Tags`**, one mark per
axis present on the case (never invented, never skipped). This is what lets
`pytest -m <mark>` slice the suite exactly like an Azure `Tag =` query would. The full
tag taxonomy (all axes, current values) lives in the project's `active/standards.md` —
read it before authoring markers; do not hardcode axis values here.

| Axis | Source tag(s) | Marker | Meaning |
|---|---|---|---|
| **1 — Lifecycle** | `Regression` | `@pytest.mark.regression` | The **critical re-run subset** — run on every change. A subset of the automated suite, not all of it. `UAT` gets no marker (it drives the client doc, not a pytest slice). |
| **1b — Execution method** | `Automation` / `Manual` | *(none)* | The automated suite itself = every case tagged `Automation`; `Manual` cases are never authored as tests. No `automation`/`manual` marker needed — a test's mere existence means `Automation`. |
| **2 — Service/Module** | e.g. `SRV`, `CMT`, `GLOBAL` (project-specific list) | `@pytest.mark.<module_lower>` (e.g. `@pytest.mark.srv`, `@pytest.mark.cmt`) | Module selector — lets a run target one feature area. |
| **3 — Platform/Surface** | `Web` / `Control_Panel` (or `IOS`/`Android` on projects that have a mobile app) | `@pytest.mark.web` · `@pytest.mark.control_panel` · `@pytest.mark.ios` · `@pytest.mark.android` | Surface selector — mirrors the Platform axis exactly. Only the values valid on the active project apply. |
| **4 — Category** | `UI` / `Compatibility` / `Auth` / `Functional-High` / `Functional-Low` / `API` / `Edge` | `@pytest.mark.<category_snake_case>` (e.g. `@pytest.mark.functional_high`, `@pytest.mark.edge`) | Category selector — e.g. run only `edge` cases after a risky change. |
| **5 — Business keyword** *(optional axis)* | e.g. `Bilingual`, `RTL`, `Workflow` (project-specific list) | `@pytest.mark.<keyword_lower>` (e.g. `@pytest.mark.bilingual`, `@pytest.mark.rtl`) | Cross-cutting selector — e.g. run every `rtl` test regardless of module. Applied only when the case actually carries a business keyword; not mandatory per test. |
| **B — Backlog traceability** *(marker-only — **not** a `Tags` axis)* | The parent PBI / backlog item ID — **not** a tag on the case. Source: the PBI ID handed to `route-automation` / `automate-test-case`, or the case's `TestedBy-Reverse` link (the same resolution `create_bug()` uses for its `PBI:<id>` tag). | `@pytest.mark.pbi_<id>` (e.g. `@pytest.mark.pbi_45231`) | Backlog selector — `pytest -m pbi_45231` runs every automated test derived from one PBI. Also feeds bug filing: `quality-control-engineer` reads the ID off this marker, so `create-azure-bug` gets the `PBI:<id>` tag and the `[<PBI ID>]` title prefix without an extra Azure lookup. |

Rules:
- A test carries **one marker per axis the case's `Tags` actually populate** — Axis 1
  (if `Regression`), Axis 2, Axis 3 (one or more), Axis 4, Axis 5 (if present). A case
  with both `Web` and `Control_Panel` tags gets both markers on its test (or, if the
  feature needed two separate tests per surface, one marker per test).
- **Marker naming:** lowercase, `snake_case`, hyphens/spaces in the tag become
  underscores (`Functional-Low` → `functional_low`). Keep the mapping deterministic so
  the same tag always produces the same marker.
- There are no `smoke` / `sanity` markers (those tags were removed from the taxonomy).
- **Axis B is mandatory on every test and is derived from the work-item link, not from
  `Tags`.** It is lettered, not numbered, precisely because the numbered axes 0–5 are the
  `Tags` taxonomy in `active/standards.md` (Axis 0 there is the MCP provenance tag
  `Ai_MCP_Injected`) — `pbi_<id>` must never be written back to a case's `Tags` or passed
  through `inject-test-cases`. Marker name is `pbi_` + the numeric ID, nothing else
  (`pbi_45231`); the argument form `@pytest.mark.pbi("45231")` is **not** used because
  `-m` cannot filter marker arguments, which loses the per-backlog run selector.
- **Also emit the backlog ID into Allure** — `allure.label("pbi", "<id>")` (alongside the
  existing title/severity) so the report groups by backlog item, not only pytest.
- **A test with no resolvable backlog ID is a defect, not a silent pass.** Record
  `NO-PBI` in the docstring, apply no `pbi_*` marker, and let the structure & redundancy
  scan flag it — mirroring the `NO-TC` rule in *Evidence file naming*.
- **Register every marker in `pytest.ini`** (no unknown-marker warnings) — scaffolding
  seeds the Axis 1/3 markers plus a placeholder block; `automate-test-case` **adds any
  new Axis 2/4/5/B marker to `pytest.ini` the first time it's used** (a `pbi_<id>` line
  per backlog item touched), so the registered list always matches what's actually
  applied in code.

---

## Automation classification pass (before injection)

Before the QA Manager injects an approved set, the surface's Automation engineer reviews
**every** case and assigns the **`Automation` / `Manual`** execution-method tag (Axis 1b
in `woqod-standards.md`) — exactly one per case, **100% coverage**, never both. **Bias
toward `Automation`:** tag `Manual` only for cases that genuinely can't be automated
(physical/hardware steps, purely visual checks, CAPTCHA, human judgement). Align each
case's `execution_type` to match. The automation build then sources **every
`Automation`-tagged case**, not just `Regression`. This pass is pure judgement — no
framework code is written and the case content is never rewritten.

---

## Reporting — Allure (mandatory)

The report must be **readable by a non-engineer** and must show, per failing step, *what
the app looked like*:

- **Allure** is the aggregator: `pytest --alluredir=reports/allure-results`, served with
  `allure serve` / `allure generate`.
- **Attachment is the deliverable, not the file.** A screenshot/video sitting in a
  folder is NOT evidence — both must land in the failing test's Allure entry via
  `allure.attach(...)` / `allure.attach.file(...)`. The two attach at different points:
  the **screenshot** in the `pytest_runtest_makereport` failure hook, the **video/
  recording** in fixture teardown (see the Video bullet — it does not exist earlier).
  Wiring that makes this work: implement `pytest_runtest_makereport` as a hookwrapper
  that stashes the call-phase report on the item (e.g. `item.rep_call`); the browser/
  driver fixture teardown reads that flag to decide whether to attach the video. A
  failing test whose Allure entry lacks its screenshot **and** video is a framework
  defect: fix the wiring before trusting the report.
- **Screenshots:** auto-captured **on every failure** (conftest hook) and on demand via
  `screenshot()`; attached to the failing Allure step (`attachment_type=PNG`).
- **Video:**
  - Web — Playwright context `record_video_dir` always on; trace
    (`tracing.start(screenshots, snapshots, sources)`) retained **on failure**.
    ⚠ Playwright only finalizes the video file when the **context closes** — attach it
    in fixture teardown *after* `context.close()`, via `page.video.path()`, guarded by
    the test's failure status.
  - Mobile — Appium screen recording (`start_recording_screen` /
    `stop_recording_screen`) around each test, attached **on failure** (retain-on-failure
    by default to save space; configurable to always-on).
- **Evidence file naming — fixed pattern, no exceptions.** Every screenshot and
  video/recording file is named
  `<type>_<date>_<time>_<testcaseID>_<project>.<ext>` — fields in that exact order:
  - `type` — `screenshot` | `video`
  - `date` — `YYYY-MM-DD`
  - `time` — `HH-MM-SS` (24h)
  - `testcaseID` — the test's QA traceability ID from its marker/docstring
    (e.g. `TAG-TOPUP-TC-014`); `NO-TC` if a test lacks one (itself a defect to fix)
  - `project` — the active project name from config (`PROJECT_NAME` in `.env` /
    `config/settings.py`)

  Example: `screenshot_2026-07-19_14-32-05_TAG-TOPUP-TC-014_WOQOD.png`. The same
  string (minus extension) is the Allure attachment name, so the report entry and the
  file on disk always match. Implemented **once** in `core/utils/reporting.py` as a
  single naming helper (e.g. `evidence_name(kind, test_case_id)`) used by all three
  capture points — the `screenshot()` wrapper, the failure-hook capture, and the
  video attach in fixture teardown (Playwright/Appium auto-generated media files are
  renamed/copied to this pattern before attaching) — never ad-hoc per test.
- **Prove it once:** after scaffolding (or any change to the failure hooks), run one
  deliberately failing probe test and open the report — confirm the screenshot and the
  video are attached to the failing entry, then delete the probe.
- **Structure the report:** use Allure `epic`/`feature`/`story` from the Service/Feature,
  `severity` from QA priority (P1→blocker … P4→minor), and `@allure.title` from the test
  case title.
- **Steps:** wrap meaningful Page-Object actions in `allure.step(...)` so the report reads
  like the manual test case's steps.

---

## Result integrity — never fake a result

> A result the framework reports must be a result the framework **actually observed**.
> Any technique that makes a test *look* green without the app truly behaving correctly is
> a defect worse than a red test — it hides breakage. This section is non-negotiable.

**Never fabricate or launder a pass:**
- **No unobserved green.** Never report a test/suite as passing unless the run was
  executed and observed. If it could not run (no device, env down, blocked precondition),
  say so explicitly — "pending environment", not "passed". Never invent Allure
  results/history, run counts, screenshots, or a pass rate.
- **No assertion tampering to force green.** Do not delete, weaken, or comment out an
  assertion; do not wrap the body in a `try/except` that swallows the failure; do not
  assert on a trivially-true condition (`assert True`, `assert 1 == 1`), and do not
  narrow an assertion until it passes. The assertion must still verify the QA case's real
  expected result.
- **No silent catch.** A caught exception that isn't re-raised or asserted on is a
  defect. Failures must surface.

**A test that catches a real product defect must FAIL — represent it honestly:**
- If a test runs and the **product** is wrong (a real bug), the correct outcome is a
  **failure (red)**, and the bug is filed (Phase 3b). Do **not** hide a live product
  defect behind a bare `@pytest.mark.xfail` or `skip` to keep the suite green — that
  masks the defect from the run summary.
- **`xfail` is allowed only for a known, already-filed product bug, and only as
  `@pytest.mark.xfail(reason="<plain English> — Bug #<id>", strict=True)`.** `strict=True`
  is mandatory: when the product is fixed the test `xpass`es → pytest turns that into a
  failure, so the marker self-cleans and can't rot. **Bare `xfail` (non-strict) is
  forbidden** — it silently absorbs both states. The `reason` must name the Bug ID.
- **`skip` / `xfail` are never a substitute for a failing assertion.** `skip` is only for
  a test that genuinely **cannot run here** (missing device/OS host, unavailable
  precondition, feature not deployed to this env) — never for one whose assertion would
  fail. Every `skip`/`xfail` carries a concrete `reason`; an unexplained one is a defect.
- **When in doubt, let it fail.** A visible red with a filed bug is always preferred over
  a quiet xfail. Do not add `xfail`/`skip` on your own judgement to tidy a run — if a
  result is inconvenient, surface it to the QA Manager, don't bury it.

**Healing touches locators, never the expected result:**
- `extract-locators` healing re-derives a **broken selector** only. The QA case's
  expected result (`expected_list` / assertion) is **not in scope** for healing — never
  edit, loosen, or reword it to match what the app currently does.
- This holds **even if the original expected result was assumed or weakly worded.** A
  weak ER is a Phase-1 review-gate defect, fixed by the QA Manager/`qa-engineer` on the
  **case**, not by the automation engineer during a locator-heal pass. Do not "fix" it
  quietly mid-heal.
- If the real assertion still fails after the locator is healed, **fail the test** — file
  the bug (Phase 3b). Do not narrow, drop, or rewrite the assertion so the healed run
  goes green.

**Reporting honesty:** the run summary you hand back states the real numbers —
passed / failed / xfailed / skipped — with the reason for every xfail/skip and the Bug ID
where one applies. Never round a mixed result up to "green".

---

## Configuration & secrets

- All environment data (base URLs per site, Appium server URL, device/OS caps, test
  credentials) comes from **env / `.env`** via `config/settings.py`. **Never** hard-code
  URLs, caps, or credentials in tests or Page Objects.
- **Web viewport default = 1920×1080 (Full HD).** Set once in the browser context
  factory (`core/web/browser.py`) — not per test — and overridable via env
  (`VIEWPORT_WIDTH` / `VIEWPORT_HEIGHT`). Responsive/mobile-web cases override it
  per test through the fixture (the browser/context fixture must accept a per-test
  viewport parameter, e.g. via indirect parametrization or a `viewport` marker), never
  by editing the default. Locator extraction inspects the page at this same default
  viewport so harvested locators match the runtime layout.
- `.env.example` is committed *inside the generated framework* (which itself is
  git-ignored here) as a template; the real `.env` is never committed anywhere. It must
  enumerate **every** value the framework reads, each with a realistic example: base URL
  per site, environment type (`ENV=dev|staging|uat|prod`), credential placeholders,
  viewport overrides, Appium server URL + device capabilities, and `PROJECT_NAME`
  (used in the evidence-file naming pattern — see Reporting above).
- **Scaffolding ends with a configuration summary.** The final reply of
  `scaffold-automation-framework` must list every configuration value the framework
  needs, pre-filled with example values, so the user knows exactly what to fill in
  (see that skill's report step).
- Default environment = **QA/UAT** (confirm with the team).

---

## Structure & redundancy scan — after every batch

After **every** batch of test-case additions or changes, run a scan of the framework
(the `automate-test-case` skill runs this as a mandatory step) and fix findings
**before** reporting done:

1. **Structure** — every Page/Screen Object and test module sits in its per-page folder
   (`pages/<page>/`, `tests/<page>/`, names mirrored between the two trees); naming
   follows the conventions above; no flat/stray files at the tree root; no
   one-file-per-test-case modules — cases for the same page merged into its single
   module.
2. **Redundancy** — no two tests cover the same case (same traceability ID, or same
   Arrange/Act/Assert intent under different names); no duplicated locator constants for
   the same element across objects; no copy-pasted Page/Screen-Object methods that
   should be a shared component object or base helper.
3. **Contract** — no raw driver imports in tests, no `sleep()`, no locators in tests,
   all markers registered in `pytest.ini`; **every test carries a marker for each tag
   axis its source case's `Tags` populate** (not just `regression`/platform — check
   Service/Module, Category, and Business-keyword marks are also present where the case
   has those tags); **every test carries its Axis B `@pytest.mark.pbi_<id>` marker** (a
   test with no `pbi_*` marker and no `NO-PBI` note in its docstring is a finding — fix
   it by resolving the backlog ID, not by deleting the note); the web browser-factory
   viewport default is still 1920×1080
   (env-overridable) — per-test overrides go through the fixture, the default is never
   edited.

Report the scan outcome explicitly: *clean*, or what was found and how it was fixed.

---

## Definition of Done (an automated test)

A test is done only when ALL hold:
- Lives in the right per-page folder of the right tree
  (`web/tests/<page>/` or `mobile/tests/<screen>/`), inside that page's **single** test
  module; imports **no** raw driver.
- All interactions go through Page/Screen Objects → wrappers; **zero `sleep()`**.
- Locators came from `extract-locators` (MCP-driven) and follow the priority order.
- Carries its QA traceability ID, its **Axis B `@pytest.mark.pbi_<id>` backlog marker**
  (plus the matching `allure.label("pbi", …)`), and **one marker per tag axis present on
  the source case** (Lifecycle/`regression`, Service/Module, Platform, Category, Business
  keyword where present) — not just `regression`/platform.
- Independent, idempotent, parallel-safe; concrete data mirrored from the QA case.
- Produces a clean Allure entry: titled, severity-tagged, steps named, screenshot+video
  attached on failure.
- The post-batch **structure & redundancy scan** is clean.
- **Result integrity holds** (see *Result integrity* above): the pass was actually
  observed, no assertion was weakened/swallowed, and any `xfail`/`skip` is
  `strict=True` with a `reason` naming its Bug ID — no bare `xfail`, no unobserved
  "green".
- Passes locally on a clean checkout (`pytest -m regression` green, or the test's own
  marker) before it's called done — a real, observed pass.

---

## Azure DevOps integration — READ enabled · post-back DEFERRED

- **READ (enabled).** The automation engineers **source the backlog from Azure**: they
  read a test suite via `mcp__azure-devops__get_test_cases_from_suite` (`plan_id`,
  `suite_id`) and build from the cases tagged **`Automation`** (the full automatable set;
  `Regression` is the critical re-run subset within it). They may still author from the
  approved chat set when no suite exists yet.
- **POST-BACK (deferred).** Writing results back to Azure (run outcomes,
  `get_test_outcome_summary`, `review_test_coverage`) stays **off** until the user
  explicitly enables it. Until then the engineers **read** cases but **write nothing** to
  Azure.
---
*Living document. Refine as the framework matures — but keep `core/` generic and keep
the repo free of the generated framework.*
