- Start Date: 2026-05-06
- RFC PR:
- Discussion Issue:
- Implementation PR(s):

# Internationalization (i18n) for DataHub UI

## Summary

This RFC proposes introducing first-class internationalization (i18n) support to DataHub's React frontend using **react-i18next** for runtime translation, **Crowdin** for translation management, **AI-assisted translation** for scale, **community language maintainers** for review and stewardship, and **CI enforcement** to keep the system correct over time.

The design keeps **English translation files in git as the source of truth** for the application. Crowdin is the operational layer for translation generation, review, and collaboration. The flow is one-directional per language: English flows from git into Crowdin; non-English translations flow from Crowdin back into git via PRs that must pass CI before merge.

This combines the strengths of a repo-native engineering workflow, AI-assisted translation, and a community-maintained localization model.

## Basic Example

A developer adds a new UI string by defining the English key in the relevant namespace file:

```json
// datahub-web-react/src/i18n/locales/en/ingest.history.json
{
  "emptyState": "No runs found for this source.",
  "statusLabel": "Status: {{status}}",
  "title": "Run History"
}
```

In the React component, the string is accessed via `useTranslation` with an explicit namespace:

```tsx
import { useTranslation } from 'react-i18next';

function RunHistoryPanel() {
  const { t } = useTranslation('ingest.history');

  return (
    <div>
      <h2>{t('title')}</h2>
      <p>{t('statusLabel', { status: run.status })}</p>
    </div>
  );
}
```

CI validates the new English keys and string usage. The English files are then synced to Crowdin, where AI-assisted translation drafts non-English values. A designated maintainer for each locale reviews those translations. Approved translations sync back to the repo as a PR and must pass CI again before merge.

## Motivation

DataHub is used globally by engineering, data, and analytics teams across many countries. The UI is currently English-only, which is a meaningful barrier for adoption in non-English markets.

Community demand has already been demonstrated by multiple independent i18n efforts:

- **datahub-project/datahub#11297** — a community contribution that introduced react-i18next infrastructure and initial translations for selected settings and group-management pages.
- **TimSchlitzer/datahub** `feat/i18n-v2` — a community branch spanning 83 commits, multiple locales, and thousands of translation keys, proving broad coverage is possible but difficult to sustain in a long-lived fork.
- Internal and community discussion showing both strong demand and the need for an upstream-supported workflow rather than repeated one-off attempts.

Prior efforts also surfaced the core implementation problems:

- DataHub's frontend is too large for a one-shot manual translation effort.
- Translation files become merge-conflict hotspots unless they are split carefully.
- Automated translation without strong context produces contaminated or inconsistent results.
- Localization only works long-term if there is clear ownership per language.

The combination of **Crowdin + AI assistance + community maintainers + CI enforcement** addresses those issues directly.

## Requirements

- Replace all user-visible hardcoded strings in `datahub-web-react` with translation keys.
- Keep **English (`en`)** in git as the permanent source of truth.
- Use **react-i18next** at runtime.
- Use **Crowdin from Phase 0** as the translation management layer.
- Use a namespace-based translation file structure with **fine-grained namespaces** for high-churn areas.
- Require **explicit namespace usage** in application code; do not rely on an implicit default namespace.
- Enforce via CI that no hardcoded JSX string literals may be merged without corresponding translation keys.
- All supported locales must maintain key parity with English.
- Translation workflow must support locale-specific plural categories, not just English-style `one` / `other`.
- AI-assisted translation must be guided by DataHub-specific terminology and review rules.
- Each supported locale must have at least one designated maintainer responsible for review and approval.
- Approved translations must sync back to the repo as PRs and pass CI before merge.
- Roll out in phases: infrastructure first, then namespace-by-namespace migration.
- New strings added during ongoing development must enter the translation pipeline immediately.
- The user's preferred locale must be persisted in their DataHub profile metadata and surfaced in Settings.

### Extensibility

- New locales must be addable without restructuring existing files.
- Namespace files must be splittable further if conflict rates remain high.
- The translation context document must evolve alongside the codebase.
- The system must support both core-team-owned and community-owned locales.
- CI must work for initial migration and steady-state development.

## Non-Requirements

- **Backend i18n**: Java, PDL schema, and backend-originated error messages remain out of scope.
- **RTL language support**: Deferred to a future RFC.
- **Fully manual translation from day one**: AI-assisted draft generation is part of the design.
- **Crowdin as runtime source**: Crowdin is not the application's runtime source of truth; git files remain canonical for shipping code.
- **Locale-aware number, date, and currency formatting**: Not in scope for this RFC.

## Detailed Design

### Core Decisions

This RFC makes the following decisions:

- Use **react-i18next** for frontend runtime i18n.
- Use **per-namespace JSON translation files in git**, sorted alphabetically.
- Use **Crowdin from the initial rollout** as the translation operations layer (the open-source tier is available at no cost for OSS projects, which fits DataHub's model).
- Use **AI-assisted translation** to generate draft translations in Crowdin, with the LLM provider configurable (Claude is the proposed default given Acryl's relationship with Anthropic).
- Require **human approval by designated locale maintainers** before any non-English translation is merged.
- Keep **CI enforcement in the repo** as the final correctness gate before merge.

### Library Selection

**react-i18next** is the recommended runtime library.

It has been validated through prior community work, supports namespace-based loading, fits DataHub's React architecture, and works well with JSON-based translation workflows.

Dependencies:

```json
{
  "i18next": "^23.x",
  "react-i18next": "^14.x",
  "i18next-http-backend": "^2.x",
  "i18next-browser-languagedetector": "^8.x"
}
```

### File Structure

```
datahub-web-react/src/i18n/
├── i18n.ts
├── locales/
│   ├── en/
│   │   ├── common.actions.json
│   │   ├── common.feedback.json
│   │   ├── entity.profile.json
│   │   ├── entity.lineage.json
│   │   ├── entity.search.json
│   │   ├── ingest.sources.json
│   │   ├── ingest.history.json
│   │   ├── govern.tags.json
│   │   ├── govern.glossary.json
│   │   ├── settings.profile.json
│   │   └── ...
│   ├── de/
│   ├── es/
│   ├── fr/
│   └── pt-br/
└── context/
    └── datahub-translation-context.md
```

English files are edited directly in the repo. Non-English files are also stored in the repo, but are primarily updated through the Crowdin workflow and synced back through PRs.

Keys must be shallowly nested or flat, and **sorted deterministically (alphabetically)** to reduce merge conflicts.

### i18next Initialization

```ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import HttpBackend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n
  .use(HttpBackend)
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    supportedLngs: ['en', 'de'],
    ns: [
      'common.actions',
      'common.feedback',
      'entity.profile',
      'entity.lineage',
      'entity.search',
      'ingest.sources',
      'ingest.history',
      'settings.profile',
    ],
    backend: { loadPath: '/locales/{{lng}}/{{ns}}.json' },
    interpolation: { escapeValue: false }, // React handles XSS
  });
```

DataHub should **not** rely on `defaultNS`. Translation usage should always be explicit so reviewers can see which namespace a string belongs to and contributors are forced to think about namespace placement.

### Namespace Strategy

A single namespace per top-level feature is too coarse for DataHub's scale. Broad files like `common.json` or `entity.json` will quickly become merge-conflict hotspots.

Instead, DataHub should use **fine-grained namespaces** based on product area and churn pattern.

| Namespace | Covers |
|---|---|
| `common.actions` | Shared actions like Save, Cancel, Delete |
| `common.feedback` | Shared error, empty, and success messages |
| `entity.profile` | Entity profile pages |
| `entity.lineage` | Lineage-related entity UI |
| `entity.search` | Search-related entity UI |
| `ingest.sources` | Source setup and management |
| `ingest.history` | Run history and execution UI |
| `govern.tags` | Tags UI |
| `govern.glossary` | Glossary UI |
| `settings.profile` | User profile settings |
| `settings.admin` | Admin settings |

This improves reviewability, ownership clarity, lazy loading, conflict isolation, and future scalability.

### Translation File Conflict Resolution

Translation JSON files are prone to conflicts when multiple PRs add keys near the same object boundary.

To reduce this, DataHub will:

- Keep namespace files small and scoped.
- Split high-churn areas more finely.
- Sort keys alphabetically.
- Preserve stable formatting in generated output.

If this is still insufficient, a future iteration can add key-level merge tooling, but that is not required initially.

### Crowdin Workflow

Crowdin is part of the initial architecture, not a future add-on.

**Role of Crowdin**

- Manages translation workflow and review.
- Provides a collaborative UI for language maintainers.
- Hosts AI-assisted translation suggestions.
- Tracks per-locale review status.
- Syncs approved translations back to the repo.

**Role of git**

- Stores the shipping translation files.
- Remains the application source of truth.
- Remains the place where CI enforcement runs.
- Remains the final gate via PR review and merge.

**Sync direction**

- English (`en/*.json`) flows **git → Crowdin** via a scheduled GitHub Action whenever English files change on `main`.
- Non-English locales flow **Crowdin → git** as automated PRs once a maintainer approves a translation in Crowdin.
- Non-English locale files in git are not edited directly; the Crowdin PR is the only path.

**High-level workflow**

1. Developers add or modify English keys in the repo.
2. Repo CI validates code and English file structure.
3. PR merges to `main`.
4. English source files sync to Crowdin.
5. Crowdin generates or updates draft translations using AI assistance.
6. Designated locale maintainers review and approve translations in Crowdin.
7. Approved translations sync back into the repo as a PR.
8. Repo CI runs again before merge.

### AI-Assisted Translation Workflow

AI is used for the heavy lifting, but is never the final authority.

The translation process is guided by a versioned **DataHub Translation Context Document**:

`src/i18n/context/datahub-translation-context.md`

It contains:

**1. Do-not-translate glossary**

Terms that must remain in English in all locales:

> DataHub, Acryl, lineage, ingestion, pipeline, schema, assertion, incident, entity, Dataset, DataFlow, DataJob, Dashboard, Chart, MLModel, GlossaryTerm, Domain, Tag, Owner, DataProduct

**2. Preferred translations per locale**

Terms that should be translated consistently across locales.

**3. Tone and formality rules**

- German: formal "Sie" (not "du")
- French: "vous" (not "tu")
- Spanish: formal register where appropriate
- PT-BR: informal "você" is acceptable

**4. Format preservation rules**

- Preserve placeholders like `{{varName}}`.
- Do not translate HTML tags or JSX attribute values.
- Preserve capitalization conventions where appropriate.

**5. Negative examples**

```
BAD:  "Are you sure you want to Entfernen this preview?"
BAD:  "Write a Beschreibung for your Ansicht."
BAD:  "DatenHub" (DataHub is a proper noun, never translate)
GOOD: "Möchten Sie diese Vorschau wirklich entfernen?"
```

**LLM provider**

The translation script and Crowdin integration must allow the LLM provider to be configured. Claude (via the Anthropic API) is the proposed default given Acryl's relationship with Anthropic, but the workflow must not be locked to a single provider.

### Pluralization Strategy

Pluralization must be treated as **locale-specific**, not English-specific.

English commonly uses `one` and `other`, but many languages require additional plural categories (`zero`, `few`, `many`, etc.) under CLDR rules. The workflow must support all plural forms required by each target locale under i18next rules.

Example English source:

```json
{
  "itemCount_one": "{{count}} item",
  "itemCount_other": "{{count}} items"
}
```

A target locale that requires more forms must contain all required categories. For example, a CLDR language that uses `one`, `few`, and `many` must produce all three:

```json
{
  "itemCount_one": "...",
  "itemCount_few": "...",
  "itemCount_many": "...",
  "itemCount_other": "..."
}
```

Requirements:

- The English source must explicitly mark pluralized strings.
- AI prompts must request all required plural forms for the target locale.
- Crowdin review must verify plural forms.
- Repo CI must validate plural category completeness per locale (using CLDR plural rule data).

### User Locale Preference

The user's chosen locale must persist across sessions and browsers. Storing it only in `localStorage` is insufficient.

**Storage**: save locale preference on DataHub user metadata, e.g. `CorpUserInfo.locale`.

**Settings UI**: add a language selector to `/settings/profile`. It lists enabled locales by their native display name (e.g., "Deutsch", "Español") alongside the language code.

**Load sequence**:

1. On app boot, fetch the authenticated user's profile.
2. If `locale` is set in profile metadata, initialize i18next with that locale.
3. Otherwise fall back to browser `navigator.language`, then to `en`.
4. When the user changes locale in Settings, update profile metadata via GraphQL and reinitialize i18next without a full reload.

**GraphQL mutation sketch**:

```graphql
mutation updateUserLocale($input: UpdateCorpUserLocaleInput!) {
  updateUserLocale(input: $input)
}
```

### Phased PR Strategy

Directly merging a broad i18n branch is not viable. The rollout should follow a phased strategy.

**Phase 0 — Infrastructure** (single PR, highest priority)

- Add i18next dependencies.
- Add `i18n.ts` initialization.
- Set up locale directory structure with first English namespace files.
- Add CI tooling (see below).
- Set up Crowdin project and bidirectional sync GitHub Actions.
- Add user locale metadata field and profile settings UI.
- Add developer documentation.
- Gate all user-visible behavior behind a feature flag (`VITE_I18N_ENABLED` — DataHub's frontend uses Vite, not Create React App).

**Phase 1–N — Per-namespace migration** (one PR per namespace or small group)

- Each PR wires `t()` calls in a module's components and adds the English namespace JSON.
- Non-English translations are generated through Crowdin and reviewed by the locale maintainer before merge.

**Ongoing development**

- New user-visible strings must add English keys in the correct namespace.
- CI enforces hardcoded string detection, locale parity, placeholder parity, and pluralization completeness.
- Crowdin sync opens follow-up PRs for newly added English keys automatically.

### Community Locale Maintainers

Each supported locale must have at least one designated maintainer from the community or core team.

Responsibilities:

- Review AI-generated translations.
- Resolve terminology disputes.
- Maintain consistency for their locale.
- Approve locale changes before sync back to git.
- Help decide whether a locale is mature enough to ship.

A locale should not be considered production-ready without an active maintainer. Locales without an active maintainer for an extended period will be marked as unmaintained and may be removed from `supportedLngs`.

## CI Tooling

Three CI scripts are added in Phase 0, plus extensions for the new requirements. CI enforcement remains critical even with Crowdin in place — Crowdin improves workflow, but CI is the final guardrail.

### 1. Hardcoded string linter — `scripts/i18n/lint-hardcoded.ts`

Uses TypeScript AST analysis (ts-morph) to detect JSX text nodes and string props that are not wrapped in a `t()` call. Runs on changed files in PRs.

### 2. Parity checker — `scripts/i18n/check-parity.ts`

Reads all locale JSON files, builds a key inventory, and fails if:

- A non-EN locale is missing keys present in EN (forward parity).
- A non-EN locale has orphan keys not present in EN (reverse parity).
- The structural nesting differs across locales for the same namespace.

### 3. Contamination detector — `scripts/i18n/check-contamination.ts`

Heuristically flags values that contain mixed-language fragments, embedded English words inside non-English sentences, or violations of the protected-term glossary.

### 4. Placeholder parity checker — `scripts/i18n/check-placeholders.ts`

For every key, the set of `{{varName}}` placeholders must be identical in EN and all non-EN locales.

### 5. Pluralization checker — `scripts/i18n/check-plurals.ts`

Validates that pluralized keys contain all CLDR plural categories required by the target locale.

### 6. Namespace coverage checker — `scripts/i18n/check-namespaces.ts`

Verifies all namespaces referenced by `useTranslation('namespace')` calls exist as JSON files for all shipped locales.

## Testing

### Static Analysis / Linting

- Hardcoded string detection.
- Namespace existence and coverage validation.
- Dead key detection (keys present in JSON but never referenced in code).

### Translation Completeness

- Forward parity from English to each locale.
- Reverse parity to prevent orphan keys.
- Structural parity per namespace.
- Plural category completeness per CLDR rules.

### Translation Quality

- Placeholder parity.
- Protected-term preservation.
- Contamination detection.
- Locale review status before sync or merge.

### Visual / Runtime Tests

- **Storybook / Playwright snapshot tests per locale**: catch layout breaks from longer strings (German strings are typically 30–40% longer than English equivalents).
- **Locale switcher E2E test**: verifies settings-based language switching.
- **Fallback-to-English test**: verifies missing keys fall back to English gracefully.
- **No raw key rendering test**: verifies keys like `entity.profile.title` do not appear literally in the rendered UI.

## How We Teach This

**Terminology**

- "translation key" (not "i18n key")
- "namespace" (not "module")
- "locale" (not "language code")
- "locale maintainer"

**Rules for developers**

- Strings live in JSON files, not in components.
- Always add English first.
- Always specify the namespace explicitly.
- Choose the narrowest reasonable namespace for a new key.
- Do not bypass the translation workflow for convenience.

**Rules for locale maintainers**

- Review AI drafts for fluency and terminology.
- Preserve DataHub-specific terms where required.
- Keep translations consistent within the locale.
- Reject low-quality or mixed-language output.

**Affected audiences**

- Frontend developers learn `useTranslation('namespace')` and the English-first workflow.
- Reviewers add a PR checklist item: "Does this PR add user-visible strings? If yes, are they wrapped in `t()` with a translation key in the right namespace?"
- Community contributors follow documented patterns for adding locales and translating namespaces.

Developer guidance lives in `datahub-web-react/i18n/CONTRIBUTING.md`.

The key mental model shift: **strings live in JSON files, not in components**. Components hold keys; JSON files hold values.

## Drawbacks

- Significant upfront migration effort across the entire DataHub React frontend.
- Added operational complexity from introducing both Crowdin and CI tooling.
- AI-generated translations still require human review.
- Locale quality depends on active maintainers; an unmaintained locale will degrade.
- Some merge conflicts may still occur in high-churn namespaces despite splitting.
- Ongoing discipline required from frontend contributors to use `t()` consistently.

## Alternatives

### Runtime Library Comparison

| Library | Format | Pros | Cons |
|---|---|---|---|
| **react-i18next** (recommended) | JSON key-value | Popular, namespace support, lazy loading, strong React fit, validated in DataHub, LLM-friendly format | Requires setup and workflow discipline |
| react-intl (FormatJS) | ICU MessageFormat | Industry-standard format, powerful pluralization and formatting | More verbose, steeper learning curve, ICU is harder for LLMs to generate reliably |
| lingui | PO / JSON | Macro-based, ergonomic | Requires Babel/SWC plugin, more build pipeline risk |
| next-intl | JSON key-value | Excellent DX | Designed for Next.js; not applicable to DataHub's Vite frontend |

**react-intl** was seriously considered. ICU's verbosity (`{count, plural, one {# item} other {# items}}`) makes LLM-assisted translation less reliable and increases the barrier for community contributors. react-i18next's simple JSON is easier to generate, review, and edit.

### Localization Workflow Alternatives

| Approach | Pros | Cons |
|---|---|---|
| **Crowdin + git + AI + maintainers** (recommended) | Best balance of scale, review UX, and repo control | More moving parts |
| Git-only manual translation | Simple architecture | Too slow and hard to scale |
| Git-only AI translation | Fast | Weak review workflow, poor community collaboration |
| Crowdin without repo CI enforcement | Good translation UX | Too risky; weak engineering guarantees |
| Lokalise instead of Crowdin | Comparable feature set | Less generous OSS tier; weaker fit for community-driven model |

### Impact of Not Doing This

Without an official plan, fragmented community attempts will continue — each solving the problem partially and inconsistently, never reaching a state where they can be merged. An Acryl-sponsored, well-scoped RFC avoids duplicated effort and gives community contributors a clear target.

## Rollout / Adoption Strategy

1. **Phase 0 PR** — infrastructure, CI tooling, Crowdin project setup, bidirectional sync, user locale metadata field, profile settings UI, explicit namespace conventions, and the `common.*` namespaces with EN + one pilot locale (DE). Gated behind `VITE_I18N_ENABLED`. No user-visible change until the flag is enabled.
2. **Phase 1–N** — one namespace migration PR at a time. Each PR is independently reviewable. Non-EN translations come through the Crowdin workflow.
3. **MVP release** — after `common.*`, `settings.profile`, and selected `entity.*` namespaces are complete, enable the locale switcher in Profile Settings for internal testing with EN + DE.
4. **Additional locales** — add locale folder, register in Crowdin, assign a maintainer, generate draft translations, review and approve, merge after CI passes.
5. **Community contribution path** — after Phase 0 merges, community contributors can volunteer as locale maintainers and submit translation PRs through the Crowdin workflow.
6. **No breaking change** — the feature flag ensures existing users see no change until i18n is production-ready. Fallback to English is always guaranteed.

## Future Work

- **Backend i18n**: Java error messages, PDL documentation fields, GraphQL mutation responses.
- **RTL language support**: Arabic, Hebrew (requires layout mirroring).
- **Translation memory and glossary expansion** in Crowdin.
- **More advanced Crowdin automation** (e.g., automatic re-translation on context document update).
- **Locale-aware formatting** for numbers, dates, and currencies via `Intl`.
- **Smarter merge tooling** if namespace splitting and alphabetical ordering prove insufficient.
- **Governance rules** for deprecating stale locales without active maintainers.

## Unresolved Questions

- What minimum activity level is required of a locale maintainer (e.g., review SLA, monthly activity)?
- What is the exact bar for shipping a new locale (coverage threshold, maintainer commitment, automated quality checks)?
- What fallback order should be used when no stored locale exists: browser locale, tenant/org-level default, or direct English?
- Which AI provider(s) should be supported in the Crowdin workflow, and how is the default chosen?
- How is maintainer ownership transferred if a community maintainer becomes inactive?
- At what conflict frequency should DataHub introduce key-level merge tooling?
- Should some shared UX copy be centralized further, or is a more distributed namespace model preferable long-term?
