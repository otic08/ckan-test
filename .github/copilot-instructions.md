# CKAN Development Guide for AI Agents

## Architecture Overview

CKAN is a Flask-based data portal platform (~2.12.0, Python ≥3.10) with a plugin-extensible architecture. All business logic flows through **action functions** (`ckan/logic/action/{get,create,update,delete,patch}.py`), not views directly. Views (`ckan/views/*.py`) and the API both call actions via `tk.get_action()`.

**Key layers**: Views/Blueprints → Logic/Actions → Models (SQLAlchemy) → PostgreSQL, with Solr for search and Redis for jobs/caching. Plugins (40+ interfaces in `ckan/plugins/interfaces.py`) can intercept any layer.

## Action Function Pattern (Critical)

Every action follows this exact structure. The `_` prefix on module-level aliases is **required** — it prevents the auto-discovery mechanism from exposing helpers as API actions:

```python
from __future__ import annotations
from ckan.logic import validate as _validate, check_access as _check_access
from ckan.types import Context, DataDict, ActionResult

@_validate(ckan.logic.schema.default_show_package_schema)
def package_show(context: Context, data_dict: DataDict) -> ActionResult.PackageShow:
    _check_access('package_show', context, data_dict)
    # implementation...
```

**Never import actions directly** — always use `tk.get_action('action_name')(context, data_dict)` to respect plugin overrides.

## Schema Convention (`@validator_args`)

In `ckan/logic/schema.py`, the `@validator_args` decorator resolves **function parameter names to validators by name**. Parameter names ARE validator lookups:

```python
@validator_args
def default_resource_schema(ignore_empty: Validator, unicode_safe: Validator, ...):
    return {"name": [ignore_empty, unicode_safe]}
```

## Type System

- Import types from `ckan.types`: `Context`, `DataDict`, `Schema`, `Validator`
- Return types: `ActionResult.PackageShow`, `ActionResult.PackageList`, etc. (see `ckan/types/logic/action_result.py`)
- `Context` is a large `TypedDict(total=False)` — a mutable state bag with 40+ optional fields
- Pyright enforced per `pyproject.toml` with granular rules (not strict mode). Excludes tests and migrations.
- Always use `from __future__ import annotations` at file top

## Plugin Development

Use `ckanext/example_*` as reference implementations (one per interface). Plugin structure:

```python
class MyPlugin(plugins.SingletonPlugin):
    plugins.implements(plugins.IActions)
    def get_actions(self):
        return {'my_action': my_action_fn}
```

**Three extension mechanisms**: replacement (last plugin wins), chaining (`@chained_action` decorator), and lifecycle hooks (`IPackageController.before_dataset_index`, etc.).

**Always use toolkit** (`import ckan.plugins.toolkit as tk`) for `get_action`, `get_validator`, exceptions, `add_template_directory`, etc.

## Import Conventions

- Use `ckan.common` for `_` (i18n), `config`, `g`, `request` — **not** Flask globals directly
- Use `ckan.plugins.toolkit as tk` for cross-cutting concerns
- `TYPE_CHECKING` guards for model imports to avoid circular dependencies

## Testing

```bash
pytest ckan/tests/                    # Core tests (config auto-loaded from pyproject.toml)
pytest ckanext/datastore/tests/       # Extension tests
```

- Tests mirror source: `ckan/tests/logic/action/test_get.py` → `ckan/logic/action/get.py`
- Fixtures registered as **pytest plugins** via `setup.cfg` entry points (not conftest.py)
- Key factories: `UserFactory`, `PackageFactory`, `ResourceFactory`, `GroupFactory`
- Plugin tests require both markers:
  ```python
  @pytest.mark.ckan_config("ckan.plugins", "my_plugin")
  @pytest.mark.usefixtures("with_plugins")
  ```
- Test infra: PostgreSQL (`ckan_test` db), Solr (port 8983), Redis (db 1). See `test-core.ini`.

## Build & Dev Commands

```bash
ckan -c development.ini run           # Dev server
ckan -c <config> db upgrade           # Alembic migrations
npm run build                         # Frontend assets (Gulp + Sass)
```

- CLI is Click-based (`ckan/cli/cli.py`), extensible via `IClick` interface
- Changelog fragments go in `changes/` using Towncrier (types: feature, bugfix, removal, migration, misc)
- Linting: Ruff (`E, F, W, C901, G, PLW0602`), line length 88

## Key Files

| Path | Purpose |
|------|---------|
| `ckan/plugins/interfaces.py` | All 40+ plugin interfaces |
| `ckan/logic/action/{get,create,update,delete,patch}.py` | Core business logic |
| `ckan/logic/schema.py` | Validation schemas with `@validator_args` |
| `ckan/types/__init__.py` | `Context`, `DataDict`, `Schema`, `Validator`, `ActionResult` |
| `ckan/config/middleware/__init__.py` | WSGI app factory (`make_app()`) |
| `ckanext/example_*` | Reference plugin implementations |
| `ckan/tests/pytest_ckan/fixtures.py` | Test fixtures and factories |
