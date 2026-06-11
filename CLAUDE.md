# CLAUDE.md

Guidance for AI assistants (Claude Code) working in this repository.

## What this project is

Paperless-ngx is a community-supported document management system: it consumes
scanned/digital documents, OCRs them, extracts metadata, and makes them
searchable through a REST API and an Angular web UI.

- Backend: Django 5 + Django REST Framework, Celery (Redis broker), Tantivy
  full-text search, optional Postgres/MariaDB/SQLite.
- Frontend: Angular (`src-ui/`), TypeScript, Bootstrap.
- Packaging: `uv` for Python deps, `pnpm` for Node deps, Docker for deployment.

## Repository layout

```
src/                   Django backend
  documents/           Core app: models, API views, consumers, classifier, search, workflows
  paperless/           Project settings, auth, celery config, shared parsers/utils
  paperless_mail/      Email-based document intake (IMAP rules, mail accounts)
  paperless_ai/        AI/LLM integration (chat, embeddings, llama-index, AI classifier)
  manage.py
src-ui/                Angular frontend
  src/app/             components/, services/, guards/, interceptors/, pipes/, directives/
  e2e/                 Playwright end-to-end tests
docs/                  User & developer documentation (built with Zensical)
docker/                Dockerfile support files, compose files, rootfs scripts
scripts/               systemd unit files, start_services.sh (spins up dev Redis/Tika/Gotenberg/db)
```

## Branching

- `main` — latest release only; no functional changes between releases.
- `dev` — target branch for all functional changes/PRs.
- `feature-X` — larger in-progress features.

PRs implementing new features should target an existing, discussed feature
request (see `CONTRIBUTING.md`). This repo's session branches (e.g.
`claude/...`) are separate from this convention and used for AI-assisted work.

## Backend (`src/`)

### Key apps
- **`documents`** — the core app. Notable modules:
  - `models.py` — `Document`, `Correspondent`, `Tag`, `DocumentType`, `StoragePath`,
    `CustomField`/`CustomFieldInstance`, `SavedView`, `ShareLink`/`ShareLinkBundle`,
    `Workflow`/`WorkflowTrigger`/`WorkflowAction`/`WorkflowRun`, `PaperlessTask`, `Note`.
  - `views.py` / `serialisers.py` / `filters.py` — DRF viewsets, serializers, django-filter filtersets.
  - `consumer.py`, `tasks.py` — document consumption pipeline (Celery tasks).
  - `classifier.py` — auto-tagging/auto-correspondent ML classifier.
  - `matching.py` — matching algorithms for tags/correspondents/types/storage paths.
  - `search/` — Tantivy-backed search (`_backend.py`, `_query.py`, `_schema.py`, `_tokenizer.py`).
  - `workflows/` — workflow trigger evaluation, actions (incl. email/webhook), mutations.
  - `plugins/` — third-party parser & date-parser plugin registries (entry-point based).
  - `management/commands/` — CLI management commands (exporter, importer, retagger,
    document_consumer, document_archiver, document_thumbnails, etc.).
- **`paperless`** — project-wide settings (`settings/`), auth backends, Celery app
  (`celery.py`), ASGI/WSGI, shared parser protocol (`parsers/`).
- **`paperless_mail`** — IMAP mail account/rule models, mail fetching and processing.
- **`paperless_ai`** — LLM-backed chat, AI document classifier, embeddings/indexing
  (llama-index + faiss), used optionally when AI features are enabled.

### REST API
- Versioned via `Accept: application/json; version=N` header
  (`DEFAULT_VERSIONING_CLASS = AcceptHeaderVersioning`).
- Current default version and allowed versions are defined in
  `src/paperless/settings/__init__.py` (`REST_FRAMEWORK["DEFAULT_VERSION"]` /
  `["ALLOWED_VERSIONS"]`) and **must** match
  `src-ui/src/environments/environment.prod.ts`. Update `docs/api.md` when bumping.
- API documentation/schema is browsable at `/api/schema/view/` (drf-spectacular).
- Document "versions" (file-level revisions) are a distinct concept from API versions —
  see `docs/api.md` for the `/api/documents/{id}/versions/...` endpoints.

### Extensibility
- Custom **document parsers** and **date-parser plugins** are loaded via setuptools
  entry points (`paperless_ngx.parsers`, `paperless_ngx.date_parsers`). See
  `docs/development.md` ("Extending Paperless-ngx") for the full parser protocol,
  required class attributes (`name`, `version`, `author`, `url`), `score()` semantics,
  and the context-manager lifecycle. Don't bypass this protocol when adding new
  format support — prefer it over hardcoding into `documents/parsers.py`.

## Frontend (`src-ui/`)

- Angular app under `src/app/`, organized by `components/`, `services/`
  (incl. `services/rest/` for API clients), `guards/`, `interceptors/`, `pipes/`,
  `directives/`, `data/`, `utils/`.
- State/UI settings flow through `services/settings.service.ts` and
  `services/profile.service.ts`.
- Localization source strings live in `src-ui/messages.xlf`; translated files in
  `src-ui/src/locale/`. Source language is `en-US`. Run `ng extract-i18n` after
  changing translatable strings.

## Development setup

1. Copy `paperless.conf.example` → `paperless.conf`, set `PAPERLESS_DEBUG=true`.
2. `mkdir -p consume media`
3. `uv sync --group dev` (installs backend deps + lint/test/docs groups)
4. `uv run prek install` (installs pre-commit hooks — runs via `prek`, a pre-commit-compatible runner)
5. From `src/`: `uv run manage.py migrate && uv run manage.py createsuperuser`
6. Start supporting services: `scripts/start_services.sh` (Docker Redis/Tika/Gotenberg/DB) or run Redis yourself.

### Running the backend (from `src/`)
```bash
uv run manage.py runserver
uv run manage.py document_consumer
uv run celery --app paperless worker -l DEBUG
```

### Running the frontend (from `src-ui/`)
```bash
pnpm install
pnpm ng serve            # dev server on :4200, proxies API to :8000
pnpm ng build --configuration production   # build for Django to serve as static files
```

## Testing

### Backend
- Run from repo root or `src/`: `uv run pytest` (configured via `[tool.pytest]` in
  `pyproject.toml`). Generates HTML/XML coverage and `junit.xml`.
- Test settings are pinned via `[tool.pytest_env]` (locmem cache, in-memory channel
  layer, fixed secret key) — don't override these when adding tests.
- Pytest markers (see `pyproject.toml`): `live`, `nginx`, `gotenberg`, `tika`,
  `greenmail`, `date_parsing`, `management`, `search`, `api`. Tests requiring
  external services use `live` plus the specific service marker.
- Test suites live under `src/documents/tests/`, `src/paperless/tests/`,
  `src/paperless_mail/tests/`, `src/paperless_ai/tests/`.

### Frontend
- Unit tests (Jest): `pnpm run test` (or `pnpm ng test`).
- E2E tests (Playwright): `pnpm exec playwright test` (UI mode: `pnpm playwright test --ui`).
- Lint: `pnpm run lint`.

### Typing (backend)
- `uv run pyrefly check src/` — baseline in `.pyrefly-baseline.json`.
- `uv run mypy src/ | uv run mypy-baseline filter` — baseline in `.mypy-baseline.txt`.
  New code should not add new mypy errors beyond the baseline.

## Code style & linting

- **Python**: formatted/linted with `ruff` (`ruff format`, `ruff check`), config in
  `pyproject.toml` (`[tool.ruff]`). Line length 88, LF line endings. `# noqa: E501`
  is acceptable for genuinely unsplittable lines.
- **TS/HTML/SCSS/Markdown**: formatted with `prettier` via pre-commit.
- **Pre-commit / `prek`**: hooks defined in `.pre-commit-config.yaml` run ruff,
  prettier, pyproject-fmt, hadolint (Dockerfile), shellcheck/beautysh (shell
  scripts), yamlfmt, codespell, and basic file hygiene checks. These run on
  commit — if a hook auto-fixes files, `git add` and retry the commit.
- Run prettier manually on changed TS files:
  `git ls-files -- '*.ts' | xargs uv run prek run prettier --files`

## Localization

- Front end: `src-ui/messages.xlf` (source), translations in `src-ui/src/locale/`.
  Adding a language touches `src-ui/angular.json`, `settings.service.ts`
  (`LANGUAGE_OPTIONS`), and `app.module.ts` (`registerLocaleData`).
- Back end: `src/locale/`. Extract with `uv run manage.py makemessages -l en_US`,
  compile with `uv run manage.py compilemessages` (compiled files are not committed).
  Adding a language also requires updating `LANGUAGES` in `src/paperless/settings/__init__.py`.
- Translations are managed via Crowdin and pushed automatically — don't hand-edit
  translated `.xlf`/`.po` files for languages other than `en-US`.

## Documentation

- Built with Zensical: `uv run zensical build` / `uv run zensical serve`.
- User-facing docs live in `docs/`; keep `docs/api.md` and `docs/configuration.md`
  in sync with backend changes (new env vars, new API versions/endpoints).

## Misc conventions

- Don't commit AI-generated code that's "mostly AI-derived" without attribution —
  see `CONTRIBUTING.md` ("AI-Generated Code").
- Django version is pinned conservatively (`django~=5.2.13`) — Django doesn't follow
  semver, so only patch bumps are safe without review.
- `.mypy-baseline.txt` and `.pyrefly-baseline.json` are large generated baseline
  files — don't hand-edit; regenerate via the respective `*-baseline` tooling if a
  refactor requires it.
