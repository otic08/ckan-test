# CKAN Development Guide for AI Agents

## Project Overview

CKAN is an open-source data portal platform built on **Flask** with a **plugin-based architecture**. The codebase is a mature Python application (~2.12.0) with strict type checking (Pyright), comprehensive testing (pytest), and extensive plugin hooks.

## Architectural Philosophy

### Flask-Based Design Principles

CKAN's Flask architecture reflects several key design decisions:

1. **Modular Application Structure**: Flask blueprints (`ckan/views/*.py`) organize routes by domain (dataset, user, admin), enabling clear separation of concerns and testability
2. **WSGI Composability**: The WSGI app created by `make_app()` can be wrapped with middleware (see `IMiddleware` interface), allowing plugins to inject Flask extensions or custom processing layers
3. **Type-Safe Configuration**: Configuration uses a declaration system (`ckan/config/declaration.py`) where plugins can register config options with validators, making config discoverable and validated at runtime
4. **Action-Based API Layer**: All business logic flows through action functions in `ckan/logic/action/`, creating a uniform API surface for both HTTP and internal calls

### Why Plugin-Based Extensibility?

CKAN's plugin system solves a critical challenge: **allowing external code to modify core behavior without forking the codebase**. This design enables:

- **Government agencies** to add custom metadata fields without changing CKAN core
- **Research institutions** to implement specialized authentication (LDAP, SAML, custom tokens)
- **Data portals** to customize authorization rules per their organizational policies
- **Third parties** to integrate with external systems (data validation services, analytics platforms)

The plugin system is built on **40+ interfaces** that expose specific extension points throughout the application lifecycle.

## Architecture

### Core Components

- **Flask Application**: Entry point via `ckan/config/middleware/__init__.py:make_app()` creates the WSGI app
- **Logic Layer**: Actions in `ckan/logic/action/{get,create,update,delete,patch}.py` provide the API surface
- **Plugin System**: `ckan/plugins/` manages extensibility via interface implementations (see below)
- **ORM Models**: `ckan/model/` contains SQLAlchemy models (Package, User, Group, Resource, ApiToken, etc.)
- **Views/Blueprints**: `ckan/views/*.py` define Flask routes (dataset, user, admin, api, etc.)
- **CLI**: Click-based commands in `ckan/cli/*.py` for server, db, config, generate, etc.

### Extension Architecture

CKAN uses a **plugin interface pattern** with 40+ interfaces (e.g., `IDatasetForm`, `IAuthFunctions`, `IActions`, `IValidators`, `ITemplateHelpers`):

```python
# Example from ckanext/example_idatasetform/plugin.py
class ExamplePlugin(plugins.SingletonPlugin, tk.DefaultDatasetForm):
    plugins.implements(plugins.IConfigurer, inherit=False)
    plugins.implements(plugins.IDatasetForm, inherit=False)
    plugins.implements(plugins.ITemplateHelpers, inherit=False)
```

Extensions in `ckanext/` follow this pattern. Use `ckanext/example_*` as reference implementations.

## How Plugins Override Core Business Logic

Plugins extend CKAN through **three primary mechanisms**: replacement, interception (chaining), and hooks.

### 1. Direct Replacement (Last-Wins)

Plugins can **completely replace** core functions by returning items with matching names. The **last plugin loaded** in `ckan.plugins` config wins:

```python
# In plugin implementing IActions
def get_actions(self):
    return {
        'package_show': my_custom_package_show,  # Replaces core package_show
        'custom_action': my_new_action            # Adds new action
    }
```

**Applies to**: `IActions`, `IAuthFunctions`, `IValidators` (with `_reverse_iteration_order=True` for validators)

**Use when**: You need to completely replace core behavior (rare - usually indicates architectural mismatch)

### 2. Interception via Chaining (Middleware Pattern)

The **chaining decorators** allow plugins to wrap existing functions, inspect/modify inputs and outputs, and optionally call the original:

```python
from ckan.plugins.toolkit import chained_action

@chained_action
def package_show(original_action, context, data_dict):
    # Pre-processing: modify inputs
    data_dict['include_custom_field'] = True
    
    try:
        # Call next in chain (or core if last)
        result = original_action(context, data_dict)
    except ValidationError as e:
        # Handle/transform exceptions
        raise CustomValidationError(e)
    
    # Post-processing: modify outputs
    result['custom_metadata'] = fetch_external_data(result['id'])
    return result
```

**Execution order**: First plugin declared → calls next → ... → core function. This creates a **middleware stack** where each plugin can:
- Transform inputs before passing to next layer
- Catch/modify exceptions
- Transform outputs before returning to caller
- Skip calling `original_action` entirely to short-circuit

**Available decorators**:
- `@chained_action` - Wrap action functions (signature: `func(original_action, context, data_dict)`)
- `@chained_auth_function` - Wrap auth functions (signature: `func(next_auth, context, data_dict)`)
- `@chained_helper` - Wrap template helpers (signature: `func(next_helper, *args, **kwargs)`)

**Use when**: You need to augment behavior (add logging, inject data, modify validation) while preserving core logic

### 3. Lifecycle Hooks (Observer Pattern)

Many interfaces provide **callback hooks** at specific lifecycle points:

```python
# IPackageController callbacks
class MyPlugin(p.SingletonPlugin):
    p.implements(p.IPackageController, inherit=True)
    
    def before_dataset_index(self, pkg_dict):
        # Modify data before Solr indexing
        pkg_dict['custom_search_field'] = extract_keywords(pkg_dict)
        return pkg_dict
    
    def after_dataset_create(self, context, pkg_dict):
        # Side effects after creation (e.g., notify external system)
        notify_external_api(pkg_dict['id'])
```

**Key hook interfaces**:
- `IPackageController`: before/after dataset operations, search modifications, indexing
- `IResourceController`: before/after resource CRUD operations
- `IGroupController`, `IOrganizationController`: before/after group/org operations
- `IDomainObjectModification`: Generic entity modification notifications

**Execution order**: Hooks are called in **plugin declaration order** (first → last)

**Use when**: You need to trigger side effects, modify data at specific lifecycle points, or integrate with external systems

### Schema Extension Pattern

`IDatasetForm` enables plugins to add custom fields to datasets:

```python
def _modify_package_schema(self, schema):
    schema.update({
        'custom_field': [tk.get_validator('ignore_missing'),
                        tk.get_validator('unicode_safe')]
    })
    return schema

def show_package_schema(self):
    schema = super().show_package_schema()
    # Add fields to API output
    schema['custom_field'] = [tk.get_converter('convert_from_extras')]
    return schema
```

**Mechanism**: CKAN uses navl-based schemas for validation. Plugins extend base schemas by adding validators/converters.

### Authorization Override Pattern

`IAuthFunctions` lets plugins implement fine-grained access control:

```python
def get_auth_functions(self):
    return {
        'package_create': custom_package_create_auth,
        'custom_action': custom_action_auth
    }

def custom_package_create_auth(context, data_dict):
    user = context.get('user')
    if user_is_in_special_group(user):
        return {'success': True}
    return {'success': False, 'msg': 'Not authorized'}
```

**Enforcement point**: Every action calls `check_access()` which looks up the corresponding auth function

### Blueprint Registration (Custom Routes)

`IBlueprint` adds custom Flask routes:

```python
def get_blueprint(self):
    from flask import Blueprint
    bp = Blueprint('my_extension', self.__module__)
    
    bp.add_url_rule('/custom-page', 'custom_page', custom_page_view)
    return [bp]
```

**Integration point**: Blueprints are registered during Flask app creation in `make_flask_stack()`

## Plugin Loading and Ordering

Plugins are loaded in the order specified in `ckan.plugins` config:

```ini
ckan.plugins = stats datastore my_custom_plugin another_plugin
```

**Critical behaviors**:
- **Actions/auth/validators**: Last plugin wins for replacements, first-to-last for chains
- **Hooks**: Executed first-to-last
- **Some interfaces use `_reverse_iteration_order = True`**: (e.g., `IConfigurer`, `IValidators`) - first plugin takes precedence
- **PluginImplementations respects order**: When iterating plugin implementations, order is preserved

**Testing with plugins**: Use pytest markers:

```python
@pytest.mark.ckan_config("ckan.plugins", "my_plugin")
@pytest.mark.usefixtures("with_plugins")
def test_my_feature():
    # Plugin is loaded and active
    result = tk.get_action('custom_action')({}, {})
```

## Critical Workflows

### Running Tests

```bash
pytest ckan/tests/  # Core tests
pytest ckanext/datastore/tests/  # Extension tests
pytest --ckan-ini=test-core.ini  # With config (set in pyproject.toml)
```

**Test patterns**:
- Use fixtures extensively (see `ckanext/datastore/tests/conftest.py`)
- Tests mirror source structure: `ckan/tests/logic/action/test_get.py` tests `ckan/logic/action/get.py`
- Markers: `@pytest.mark.ckan_config`, `@pytest.mark.usefixtures("with_plugins")`

### Development Server

```bash
ckan -c development.ini run  # Development mode
# Or via test config:
ckan -c test-core.ini run
```

### Frontend Build

```bash
npm install  # First time
npm run build  # Build assets (Gulp + Sass)
npm run watch  # Watch mode for development
```

**Asset structure**: `ckan/public/` (compiled), `ckan/public/base/scss/` (source SCSS), `ckan/public/base/javascript/` (JS modules)

### Database Migrations

CKAN uses **Alembic** (see `ckan/migration/`):

```bash
ckan -c <config> db upgrade  # Run migrations
ckan -c <config> db downgrade  # Rollback
```

### Configuration

- Test config: `test-core.ini` (PostgreSQL, Solr, Redis)
- Type-safe config via `CKANConfig` and declaration system (`ckan/config/declaration.py`)
- Plugins declare config via `IConfigDeclaration` interface

## Project-Specific Conventions

### Type Hints

**Pervasive typing** enforced by Pyright (see `pyproject.toml`):
- Use `from ckan.types import Context, DataDict, Schema, Validator`
- Actions return `ActionResult.*` types
- Models use SQLAlchemy 2.0 `Mapped[]` annotations

### Logic Layer Patterns

**Action functions** follow a strict pattern:

```python
from ckan.logic import validate, check_access, NotFound, ValidationError

@validate(ckan.logic.schema.default_pagination_schema)
def package_list(context: Context, data_dict: DataDict) -> ActionResult.PackageList:
    check_access('package_list', context, data_dict)
    # Implementation...
```

Call actions via `toolkit.get_action('action_name')(context, data_dict)`.

### Template Helpers

Add helpers via `ITemplateHelpers`:

```python
def get_helpers(self):
    return {'country_codes': country_codes_helper}
```

Access in templates: `{{ h.country_codes() }}` (helpers available as `h`)

### Plugin Toolkit

**Always use toolkit** (`ckan.plugins.toolkit` or `import ckan.plugins.toolkit as tk`):

```python
tk.get_action('package_show')
tk.get_validator('not_empty')
tk.NotAuthorized  # Exception classes
tk.add_template_directory(config, 'templates')
```

### Authentication Patterns

- **API Tokens**: JWT-based (see `ckan/lib/api_token.py`, models in `ckan/model/api_token.py`)
- Extension point: `IApiToken` interface for custom token handling (see `ckanext/example_iapitoken/`)
- Authorization checks via `ckan.authz` and action-specific auth functions in `ckan/logic/auth/`

### Validation

Schema-based validation using `ckan.lib.navl.dictization_functions`:

```python
schema = {
    'name': [tk.get_validator('not_empty'), tk.get_validator('unicode_safe')],
    'email': [tk.get_validator('ignore_missing'), tk.get_validator('email_validator')]
}
```

Decorate actions with `@validate(schema)`.

### Signals

Use `ckan.lib.signals` for event-driven patterns (e.g., `after_dataset_create`). See `ckanext/example_isignal/`.

## Key Files Reference

- **Plugin interfaces**: `ckan/plugins/interfaces.py` (40+ interfaces like IDatasetForm, IResourceController)
- **Plugin core**: `ckan/plugins/core.py` (PluginImplementations, load/unload)
- **Action layer**: `ckan/logic/action/{get,create,update,delete,patch}.py`
- **Schema definitions**: `ckan/logic/schema.py`
- **CLI main**: `ckan/cli/cli.py` (registers all commands)
- **WSGI app factory**: `ckan/config/middleware/__init__.py:make_app()`

## Testing Dependencies

- PostgreSQL (localhost:5432, database: `ckan_test`)
- Solr (localhost:8983/solr/ckan)
- Redis (localhost:6379/1)

See `test-core.ini` for full config.

## Common Pitfalls

1. **Don't bypass the toolkit**: Use `tk.get_action()` not direct imports
2. **Context is critical**: Always pass `context` dict with `user`, `model`, etc.
3. **Plugin loading**: Enable plugins in config (`ckan.plugins = stats datastore`) and use `@pytest.mark.usefixtures("with_plugins")` in tests
4. **Schema validation**: Actions fail silently without `@validate(schema)` decorator
5. **Template paths**: Use `tk.add_template_directory()` in plugin's `update_config()` method
6. **Type checking**: Code must pass Pyright strict checks per `pyproject.toml` config

## Documentation

- Official docs: https://docs.ckan.org
- API docs built from docstrings (Sphinx in `doc/`)
- Extension tutorial: `doc/extensions/tutorial.rst`
- Frontend guidelines: `doc/contributing/frontend/index.rst`
