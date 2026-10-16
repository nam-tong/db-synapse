# DB Synapse

**Static ER diagrams, N+1 detection, migration-risk checks and schema drift detection for Django — without booting it.**

DB Synapse parses `models.py` and `migrations/` as source code. There is no database connection, no `runserver`, no `DJANGO_SETTINGS_MODULE` and no working virtualenv required. The same parser core ships three ways: a VS Code extension, a Python CLI, and an MCP server for AI agents.

[![CI](https://img.shields.io/github/actions/workflow/status/nam-tong/db-synapse/ci.yml?branch=main&label=CI&logo=github)](https://github.com/nam-tong/db-synapse/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/nam-tong/db-synapse?color=16a34a)](LICENSE)

> [!NOTE]
> **This project is a fork.** DB Synapse is a fork of
> [**FROWNINGdev/orm-synapse**](https://github.com/FROWNINGdev/orm-synapse) — originally published as *Django ORM Lens*,
> later *ORM Synapse* — created and MIT-licensed by [FROWNINGdev](https://github.com/FROWNINGdev). All of the original
> parser, rules, CLI, MCP server and VS Code extension are their work; the original copyright and licence are kept
> unchanged in [LICENSE](LICENSE).
>
> This fork is maintained independently by [@nam-tong](https://github.com/nam-tong). It is **not affiliated with or
> endorsed by** the original author. It renames the project to *db-synapse* (package `db_synapse`, settings namespace
> `dbSynapse`, comment directive `db-synapse-disable`) and continues development from there.
>
> Marketplace, Open VSX, PyPI, Smithery and other registry listings, download counts, stars and
> press mentions that belong to the upstream project are **not** claimed here. See [Install](#install) for how to get
> this fork, and [Credits](#credits-and-upstream) for the full lineage.

---

## Features

### In the editor (VS Code extension)

- **Model tree** — every app → model → field → `Meta` option, grouped by application, with field-type icons. Filter by app, model or field name; toggle apps or models to drop them from the diagram. Split `models/` packages and `abstract_models.py` are supported.
- **Live ER diagram** — a Mermaid entity-relationship diagram of the whole schema that redraws as you edit. `ForeignKey`, `OneToOneField` and `ManyToManyField` become cardinality arrows with edge labels (`CASCADE`, `SET_NULL`, `through Model`, `related_name`). Themes: `auto` / `default` / `dark` / `forest` / `neutral`; export to SVG.
- **Hover cards and CodeLens** — hover a `ForeignKey('app.Model')` for the target model's fields, relations and a jump link; a CodeLens above each model shows field count, relation count and an *Open ER diagram* action.
- **Field completion** inside `.filter()`, `.exclude()` and `.get()`.
- **Inline diagnostics and QuickFixes** — 19 Ruff-style rules (`DOL001`–`DOL041`) with per-rule severity, Clippy-style fix applicability, and inline suppression. Examples: `.count() > 0` → `.exists()`, `null=True` on `CharField`, missing `on_delete`, `datetime.now()` → `timezone.now()`.
- **Impact analysis** — right-click a field or model to find every reference grouped by Django layer (models, serializers, forms, admin, views, urls, templates, tests, migrations), tagged *Certain / Likely / Possibly*.
- **Time-travel schema diff** — pick two commits and get a typed markdown diff (`AddModel` / `DropModel` / `RenameModel` / `ModifyModel`) with confidence-scored rename detection and `UniqueConstraint` tracking.
- **Query builder** — right-click a field or model to insert an ORM snippet (`select_related` for FKs, `annotate` honouring `related_name`, `prefetch_related` for M2M, …).
- **factory_boy generator** — scaffold a `DjangoModelFactory` with Faker providers chosen from the field types.

### On the command line (Python CLI)

| Command | What it does |
|---|---|
| `scan` | Every app, model and field (`-f table` / `json`) |
| `describe <app.Model>` | One model as Markdown |
| `list` | Flat `app.Model` list for piping |
| `er` | ER diagram as Mermaid, DBML, D2, PlantUML or Graphviz `dot` |
| `diff a.json b.json` | Structural diff of two schema dumps |
| `nplusone` | Static N+1 detection, followed across function boundaries |
| `migration-risk` | 16 rules for risky migration operations (NOT NULL without default, table-locking index builds, irreversible data migrations, …) |
| `blast-radius` | Migration risks joined with the code that still reads the affected schema and the cascade fallout |
| `drift` | `makemigrations --check` without booting Django: migrations replayed and compared with `models.py` |
| `impact <name>` | What still references a model or field |
| `cascade <app.Model>` | What one `delete()` takes down |
| `suggest-indexes <app.Model>` | `Meta.indexes` proposals from observed QuerySet usage |
| `signals` | Sender → signal → handler graph |
| `migration-deps` | Per-app migration dependency graph (text / json / mermaid) |
| `stats-sql` | Read-only SQL you run yourself to feed `blast-radius --stats` with real row counts |

```bash
db-synapse scan -f table
db-synapse er -f dbml > schema.dbml
db-synapse nplusone --format github
db-synapse migration-risk -f sarif --exit-zero > findings.sarif
db-synapse blast-radius -f markdown
```

Every command accepts `--path <dir>` and `--exclude <glob>`. Output formats include `github` (PR annotations), `sarif` (Code Scanning) and `markdown` (postable PR comment). Exit codes are CI-native: `diff` and `nplusone` exit `1` on findings, `migration-risk` and `blast-radius` on critical findings, `drift` when a field is declared but never migrated. Add `--exit-zero` for report-only runs. The CLI has no runtime dependencies and needs Python 3.9+.

### For AI agents (MCP server)

`db-synapse-mcp` exposes thirteen read-only tools so an agent answers schema questions from the parsed source instead of guessing:

`list_apps` · `list_models` · `describe_model` · `find_relations` · `cascade_preview` · `er_diagram` · `describe_migration_dependency` · `suggest_indexes` · `signal_graph` · `blast_radius` · `drift` · `impact` · `nplusone_scan`

```jsonc
{
  "mcpServers": {
    "db-synapse": {
      "command": "db-synapse-mcp",
      "env": { "DJANGO_ORM_LENS_ROOT": "/abs/path/to/your/django/project" }
    }
  }
}
```

The workspace is resolved from the tool call's `workspace_root` argument, then `$DJANGO_ORM_LENS_ROOT`, then the current directory. Paths that are not Django projects return a structured `WORKSPACE_NOT_DJANGO` error. Restrict where the server may read with `DJANGO_ORM_LENS_ALLOWED_ROOTS` (`:`-separated, `;` on Windows).

> The two environment variable names still carry the project's earliest name (*Django ORM Lens*). They are unchanged so existing configs keep working.

### In CI

```yaml
# Pull-request schema review: one comment, updated in place on every push
- uses: nam-tong/db-synapse@action-v1
  with:
    command: blast-radius
    only-changed: true
    comment: true
    github-token: ${{ github.token }}
```

```yaml
# pre-commit
repos:
  - repo: https://github.com/nam-tong/db-synapse
    rev: py-v1.13.3
    hooks:
      - id: db-synapse-nplusone
      - id: db-synapse-migration-risk
```

SARIF output can be uploaded with `github/codeql-action/upload-sarif` to populate the repository's Security tab. The Action installs the CLI from PyPI unless you pass `install: false`; [`schema-review.yml`](.github/workflows/schema-review.yml) shows how this repository runs it from source.

---

## Install

This fork is not published to the VS Code Marketplace, Open VSX or PyPI under the `db-synapse` name. Install from source:

```bash
git clone https://github.com/nam-tong/db-synapse && cd db-synapse
```

**CLI and MCP server**

```bash
pip install ./cli              # CLI
pip install "./cli[mcp]"       # CLI + MCP server
```

**VS Code extension**

```bash
npm ci
npm run build
npm run package                # produces db-synapse-<version>.vsix
code --install-extension db-synapse-*.vsix
```

**Docker**

```bash
docker build -t db-synapse .
docker run --rm -v "$PWD:/workspace" db-synapse scan --path .
```

Works in VS Code, Cursor, Windsurf and VSCodium (install the VSIX), and with any MCP-compatible client (Cursor, Claude Code, Aider, Zed, Continue).

---

## Configuration

```jsonc
// .vscode/settings.json
{
  "dbSynapse.excludeGlobs": ["**/migrations/**", "**/node_modules/**", "**/venv/**", "**/.venv/**"],
  "dbSynapse.autoRefresh": true
}
```

| Setting | Default | What it does |
|---|---|---|
| `dbSynapse.excludeGlobs` | see above | Glob patterns skipped when scanning |
| `dbSynapse.autoRefresh` | `true` | Rescan when `models.py` changes |
| `dbSynapse.codeFixes.enabled` | `true` | Master switch for diagnostics and QuickFixes |
| `dbSynapse.rules` | `{}` | Per-rule severity, e.g. `{ "DOL007": "off" }` |
| `dbSynapse.rulesSelect` | `[]` | Ruff-style select, e.g. `["DOL0"]` |
| `dbSynapse.rulesIgnore` | `[]` | Ruff-style ignore, e.g. `["DOL03"]` |
| `dbSynapse.dbEngine` | `auto` | `auto` / `postgres` / `mysql` / `sqlite` — engine-specific rules follow the chosen engine |

The full list is in `package.json` under `contributes.configuration`.

### Suppressing a rule

```python
# db-synapse-disable-next-line DOL007
for user in User.objects.all():
    print(user.profile)

qs.count() > 0  # db-synapse-disable-line DOL001

# db-synapse-disable DOL011   <- on its own line: off for the rest of the file
```

### Commands

Open the command palette and type **DB Synapse**: Refresh, Show ER Diagram, Filter Models, Clear Filter, Find Reverse References, Generate factory_boy Factory, Schema Diff (Time-Travel), Find Impact, Build Query.

---

## Rules

Nineteen editor rules (`DOL001`–`DOL041`), sixteen migration-risk rules and the static N+1 analyzer. Each has a page with bad/good examples and QuickFix behaviour — start at [docs/rules/README.md](docs/rules/README.md).

| Category | Codes |
|---|---|
| Queryset | `DOL001`–`DOL007` |
| Model definition | `DOL011`–`DOL016` |
| Datetime | `DOL021`–`DOL022` |
| Forms / views | `DOL031`–`DOL032` |
| Raw SQL | `DOL041` |
| [Migration risks](docs/rules/migrations.md) | 16 rules |
| [Static N+1](docs/rules/nplusone.md) | 1 analyzer |

---

## Development

```bash
npm ci && npm test                       # extension: build + node --test
cd cli && pip install -e ".[mcp]" pytest && python -m pytest
```

The TypeScript and Python parsers are kept in step by shared golden fixtures (`cli/tests/fixtures/golden`), which parse vendored models from Zulip, Saleor, Wagtail, django CMS, Mezzanine and Read the Docs. Release history is in [CHANGELOG.md](CHANGELOG.md); contribution notes in [.github/CONTRIBUTING.md](.github/CONTRIBUTING.md).

---

## Credits and upstream

- **Original project:** [FROWNINGdev/orm-synapse](https://github.com/FROWNINGdev/orm-synapse) — © 2026 FROWNINGdev, MIT. Development history before the fork, including contributions from community members, lives there and in this repository's git log and [CHANGELOG](CHANGELOG.md).
- **This fork:** [nam-tong/db-synapse](https://github.com/nam-tong/db-synapse) — renamed and maintained independently.
- Issues about this fork belong in [this repository's tracker](https://github.com/nam-tong/db-synapse/issues), not upstream's.

## License

[MIT](LICENSE) — © 2026 FROWNINGdev (original work). Modifications in this fork are released under the same licence.
