---
type: authoring workflow
title: Versioned Content
description: Author shared documentation that the build emits as Python and JavaScript variants. This workflow covers source ownership, conditional content, language-aware links and snippets, navigation, redirects, release-claim checks, and output verification.
tags: [versioning, conditional-rendering, markdown, snippets, package-validation, navigation]
verified:
  - by: openwiki/0.4.3
    at: 2026-09-20T08:19:24.227Z
sources:
  - id: openwiki-source-ddbddbe474c8dc57119458d7
    resource: repo://.agents/skills/docs-code-samples/SKILL.md
  - id: openwiki-source-012f2c78e3b1446dfc35803f
    resource: repo://Makefile
  - id: openwiki-source-d0cdf44431684bdedf34705a
    resource: repo://pipeline/core/builder.py
  - id: openwiki-source-17f3856bce97f37118963062
    resource: repo://pipeline/preprocessors/handle_auto_links.py
  - id: openwiki-source-06a4c757b1153b7de4f47a0e
    resource: repo://pipeline/preprocessors/markdown_preprocessor.py
  - id: openwiki-source-99b53585619b83f258314f8b
    resource: repo://scripts/check_version_claims.py
  - id: openwiki-source-a9a8730b7e43a5ad2d0af4f1
    resource: repo://src/docs.json
  - id: openwiki-source-97e34e6957c53e95a26c2e05
    resource: repo://src/oss/deepagents/quickstart.mdx
  - id: openwiki-source-24e5f74f0f40e9bfd381871f
    resource: repo://tests/unit_tests/test_builder.py
  - id: openwiki-source-607673c5c40214b511f9e0a7
    resource: repo://tests/unit_tests/test_check_version_claims.py
generated: { by: "openwiki/0.4.3", at: "2026-09-20T08:19:24.227Z" }
---

# Versioned Content

Versioned documentation keeps shared prose in one source while the build emits language-specific artifacts. Make three independent decisions: **source ownership** determines where the authored file belongs, **emitted routes** determine what the builder writes, and **visible navigation and redirects** determine how readers discover or reach those routes. Never edit `build/`; `make build` regenerates it.

```mermaid
flowchart TD
    Start["Choose the authored source"] --> Ownership{"Which source class"}
    Ownership -->|Shared OSS| Shared["Place below src/oss"]
    Ownership -->|Language-only OSS| Specific["Place below src/oss/python or src/oss/javascript"]
    Ownership -->|Language-agnostic product| Unversioned["Place in an unversioned product directory"]
    Ownership -->|Managed Deep Agents| MDA["Place managed-deep-agents MDX in src/langsmith"]
    Shared --> Dual["Emit Python and JavaScript OSS routes"]
    Specific --> One["Emit the matching language route"]
    Unversioned --> Default["Emit one unprefixed route using Python target"]
    MDA --> MDADual["Emit Python and JavaScript LangSmith routes"]
    Dual --> Nav["Configure navigation in src/docs.json"]
    One --> Nav
    Default --> Nav
    MDADual --> Nav
    Nav --> Redirects["Add intentional legacy redirects"]
    Redirects --> Inspect["Build and inspect every expected output"]

    classDef process fill:#E5F4FF,stroke:#006DDD,stroke-width:2px,color:#030710
    classDef decision fill:#FDF3FF,stroke:#7E65AE,stroke-width:2px,color:#504B5F
    classDef output fill:#EBD0F0,stroke:#885270,stroke-width:2px,color:#441E33
    class Start,Shared,Specific,Unversioned,MDA,Nav,Redirects process
    class Ownership decision
    class Dual,One,Default,MDADual,Inspect output
```

This source-backed routing flow separates authored location from generated routes and reader-facing configuration.

## Select source ownership and routes

Choose the authored path based on ownership, not a desired URL. The builder maps its `js` target to `javascript` in public paths.

| Source ownership | Authored location | Emitted route or routes |
| --- | --- | --- |
| Shared OSS page | Most content below `src/oss/`, such as `langchain/`, `langgraph/`, and `deepagents/` | `/oss/python/...` and `/oss/javascript/...` |
| Language-only OSS page | `src/oss/python/...` or `src/oss/javascript/...` | The matching `/oss/python/...` or `/oss/javascript/...` route only |
| Language-agnostic product | `src/oss/openwiki/...` or `src/oss/deepagents/code/...` | One unprefixed `/oss/openwiki/...` or `/oss/deepagents/code/...` route |
| Managed Deep Agents page | A `.mdx` file named `managed-deep-agents*.mdx` directly in `src/langsmith/` | `/langsmith/python/...` and `/langsmith/javascript/...` |
| Other LangSmith page | `src/langsmith/...` | Its ordinary unprefixed LangSmith route, processed with the Python target |

Most OSS sources are emitted into Python and JavaScript route trees. `oss/deepagents/code` and `oss/openwiki` are deliberate exceptions: each is rendered once with the Python conditional target. Thus an unqualified OSS link from one of those products still resolves to Python, but links to that product's own root remain unprefixed.

Do not create copies under generated `python` or `javascript` paths. For ownership guidance, see [Source directory map](/openwiki/architecture/source-map.md).

## Write conditional content

Use one shared source for common text and sequential `:::python` and `:::js` blocks only where content differs. Conditional blocks create separate Python and JavaScript outputs from that shared source: the selected supported-language block is emitted without its markers, the other supported block is removed, and content outside both blocks remains in each artifact.

````markdown
Shared explanation.

:::python
```python
from langchain.agents import create_agent
```
:::

:::js
```typescript
import { createAgent } from "langchain";
```
:::
````

Keep neutral headings and shared explanations outside branches so both rendered pages remain coherent. Conditional rendering accepts only `python` and `js` target keys. Unsupported labels are preserved unchanged; an invalid target raises `ValueError`.

### Scope API references to the branch

`@[Name]`, `@[title][Name]`, and backticked forms are resolved before conditional rendering. Autolink replacement tracks the active `:::language` scope outside ordinary code fences, resolves known references from that scope, and logs rather than fails when a reference is absent. Put language-dependent references inside their matching branches:

````markdown
:::python
See @[StateGraph].
:::

:::js
See @[StateGraph].
:::
````

Run `make check-cross-refs` after adding or changing an API reference. A rendered page can hide an unresolved reference in the branch excluded from one artifact. See [Markdown preprocessing pipeline](/openwiki/concepts/preprocessing.md).

### Avoid parser traps

Conditional rendering is regex-based, not code-fence-aware or nested-block-aware. Do not put live conditional markers inside a normal code fence and do not nest branches. Escape both literal markers when documenting the syntax:

````markdown
\:::python
This is displayed literally.
\:::
````

Unsupported labels and unmatched eligible openings remain unchanged, so they are not validation or nesting constructs.

## Author links and snippets for the active variant

For a destination that should follow the current OSS language, author an absolute, unqualified `/oss/...` link:

```mdx
<!-- openwiki: broken internal link [/oss/langgraph/overview] file "/oss/langgraph/overview" does not exist. Fix the href or restore the target, then delete this comment. -->
[LangGraph overview](/oss/langgraph/overview)
```

For a target-language build, unqualified absolute `/oss/` links receive the target route segment. Already-qualified routes, image paths, and the OpenWiki and Deep Agents Code roots are preserved. This applies to Markdown links and HTML `href` attributes. Use a qualified route only when the destination must remain fixed to that language.

Bare Managed Deep Agents links behave similarly: a `/langsmith/managed-deep-agents...` reference is rewritten to the selected `/langsmith/python/...` or `/langsmith/javascript/...` route. An already-qualified Managed Deep Agents route remains fixed.

Store reusable Markdown or MDX fragments under `src/snippets/` and import them without a language prefix from a versioned page:

```mdx
import RequiresLanggraphServer from '/snippets/oss/requires-langgraph-server.mdx';
```

Versioned-page imports of unqualified Markdown snippets are rewritten to `/snippets/python/` or `/snippets/javascript/`. Already scoped imports and JSX or TSX component imports are unchanged. Use absolute `/oss/...` links inside a shared snippet rather than fixed relative paths.

Markdown snippets are independently preprocessed and emitted in Python and JavaScript copies plus a Python-targeted default copy. The absolute links in those copies work for consumers at arbitrary nesting depths. Use separate language-specific snippet components when the reusable unit itself differs; for example, the shared Deep Agents quickstart imports Python and JavaScript components separately and renders each in its matching conditional branch.

### Generate runnable examples instead of hand-maintaining sample snippets

For an executable example, make `src/code-samples/` the source of truth. Put the reader-visible region between language-suffixed `:snippet-start:` and `:snippet-end:` markers, and put test-only setup or assertions in a trailing `:remove-start:` block. The test must execute the visible snippet before it exits.

Test the changed sample, then regenerate its derived MDX:

```bash
make test-code-samples FILES="src/code-samples/langchain/return-a-string.py"
make code-snippets
```

`make code-snippets` extracts tagged source into the gitignored `src/code-samples-generated/` intermediate directory and generates MDX components under `src/snippets/code-samples/`. Import generated Python and JavaScript components after a page's frontmatter and render each in its matching branch. Do not hand-edit derived snippet MDX: change the sample and regenerate it. Keep related Python snippets in one source file when practical; split TypeScript snippets when top-level imports or bindings would collide during one-file execution.

## Check package-version claims before publishing

A package floor or pin promises that readers can resolve the named release. Establish from the owning product source or changelog that a version is the actual minimum, then check that the written release was published:

```bash
uv run python scripts/check_version_claims.py --files src/path/to/page.mdx
```

The checker scans `.mdx` pages for `>=` floors and `==` pins. It chooses PyPI or npm from package syntax, nearby language labels, conditional fences, source paths, and then a PyPI fallback; it checks only whether the written release was published. A pass proves availability, not that the release introduced the feature. Registry lookup failures and invalid lookup inputs are unresolved rather than unpublished-version failures. Exact matches and shortened version-series floors pass when a corresponding published release exists. Correct a bad requirement rather than adding it to `scripts/version_claims_ignore.txt` unless it is a reviewed, intentional exception.

## Configure navigation and redirects separately

A source file and a built route do not create a visible navigation entry. For a new or moved page, update the appropriate `src/docs.json` product, menu, language dropdown, tab, and group after confirming its output route. Managed Deep Agents has parallel Python and JavaScript route entries in separate language dropdowns. `docs.json` maps unprefixed and legacy Managed Deep Agents URLs to Python destinations.

A redirect is independent from source placement and navigation. Add or retain a `redirects` entry only for a supported legacy or alias URL, and point it at the intended canonical route. The builder emits no unprefixed Managed Deep Agents pages; old unprefixed URLs redirect to Python routes. Do not create an unversioned source copy merely to serve an old URL.

When a page moves, decide all three outcomes explicitly:

1. Keep, move, or split the authored source according to ownership.
2. Confirm every emitted Python, JavaScript, or unprefixed route that must exist.
3. Update navigation entries and add redirects for old public routes when readers need continuity.

For the navigation procedure, see [Adding and modifying documentation pages](/openwiki/operations/adding-pages.md).

## Build and inspect outputs

Run a clean build after changing a shared source, link, snippet, route rule, navigation entry, or redirect:

```bash
make build
```

The full build clears the build directory, generates OSS Python and JavaScript variants, builds unversioned OSS products and LangSmith content, emits Managed Deep Agents variants, and then copies shared files. It also copies installed sandbox snippet components and generates `llms.txt` files. Inspect generated output as verification only; do not edit it.

For a shared OSS or Managed Deep Agents page, inspect both language variants. For a language-only or unversioned product page, inspect the expected route and confirm unexpected language duplicates do not exist.

Check the following:

1. The expected route exists, with matching conditional content and without the opposite supported branch or selected-block markers.
2. Shared prose appears in every expected variant.
3. Conditional API references resolve in the correct scope.
4. Unqualified OSS links, Managed Deep Agents links, and Markdown snippet imports have the expected language prefix.
5. Intentional fixed-language links, image paths, and unversioned OpenWiki or Deep Agents Code paths remain unchanged.
6. Each new route appears in the intended navigation location, and each old route has either a deliberate redirect or a documented removal decision.

When changing builder behavior, add a focused regression in `tests/unit_tests/test_builder.py`. Existing coverage exercises OSS prefix insertion and exemptions, unversioned product output, language-scoped snippets, and Managed Deep Agents dual routes. Test changes to package-version parsing in `tests/unit_tests/test_check_version_claims.py`, including registry-selection precedence and lookup failures. For fence-focused tests, see [Conditional rendering tests](/openwiki/testing/conditional-rendering.md).

## Checklist

- [ ] Choose the authored source class before choosing navigation or redirects.
- [ ] Keep shared prose outside sequential `:::python` and `:::js` branches.
- [ ] Do not nest conditionals or rely on code fences to protect live markers.
- [ ] Use unqualified `/oss/...` links only when the destination should follow the active language.
- [ ] Import Markdown snippets unprefixed and use absolute OSS links in shared snippets.
- [ ] Test and regenerate executable code samples from `src/code-samples/`; do not hand-edit derived snippets.
- [ ] Establish each package floor from its owning source and run `check_version_claims.py` for changed MDX with specifiers.
- [ ] Configure navigation and redirects in `src/docs.json` separately from source and output routes.
- [ ] Run `make build`, then inspect every expected output without modifying generated artifacts.

## See also

- [Language versioning strategy](/openwiki/concepts/versioning.md)
- [Markdown preprocessing pipeline](/openwiki/concepts/preprocessing.md)
- [Source directory map](/openwiki/architecture/source-map.md)
- [Conditional rendering tests](/openwiki/testing/conditional-rendering.md)
- [Code sample lifecycle](/openwiki/workflows/code-sample-lifecycle.md)
- [Adding and modifying documentation pages](/openwiki/operations/adding-pages.md)
