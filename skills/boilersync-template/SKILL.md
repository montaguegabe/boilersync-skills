---
name: boilersync-template
description: >-
  Use this skill when users ask about BoilerSync template creation,
  template syntax, .starter or .boilersync files, template inheritance,
  or BoilerSync init/pull/push workflows.
version: 0.1.0
---

# BoilerSync Template

BoilerSync scaffolds projects from templates and keeps template and project evolution in sync with `init`, `pull`, and `push`.

## Core Workflow

```bash
# Clone template source repo into local cache
boilersync templates init https://github.com/your-org/your-templates.git

# Initialize a project from a source-qualified template
boilersync init your-org/your-templates#python/service-template

# Initialize a well-defaulted template without prompts
mkdir my-app-workspace
cd my-app-workspace
boilersync init your-org/your-templates#python/service-template --non-interactive

# Pull template updates later
boilersync pull

# Push committed project changes back to template source
boilersync push
```

## CLI Essentials

- `boilersync init TEMPLATE_REF`: first-time project generation (empty target directory)
- `boilersync init TEMPLATE_REF --non-interactive`: generate without prompts when the template supplies defaults for every required variable
- `boilersync pull [TEMPLATE_REF]`: apply template changes to an existing project
- `boilersync push`: promote committed project changes back to template source
- `boilersync templates init`: clone/register template source repos into local cache
- `boilersync templates details TEMPLATE_REF [--json]`: inspect effective template inputs after inheritance is resolved

## Non-Interactive Init (Agents/CI)

`boilersync init` supports non-interactive mode via `--non-interactive` (alias: `--no-input`).

Before unattended init, inspect the template inputs:

```bash
boilersync templates details your-org/your-templates#python/service-template --json
```

Use this pattern when an AI agent or CI job must never block on prompts:

```bash
boilersync init your-org/your-templates#python/service-template \
  --non-interactive \
  --var name_snake=my_service \
  --var name_pretty="My Service" \
  --var author_name="Jane Doe" \
  --var author_email="jane@example.com"
```

Rules:

- Include `--non-interactive` for unattended execution.
- Pass project naming through normal template variables such as `name_snake` and `name_pretty`.
- Provide required template variables with one or more `--var KEY=VALUE` flags.
- Prefer `template.json` `defaults` for values that can be derived from project naming, so agents and CI can use fewer `--var` flags.
- If a required variable is missing in non-interactive mode, BoilerSync exits with an error instead of prompting.

## Template References

Preferred source-qualified references:

- `org/repo#subdir`
- `https://github.com/org/repo#subdir`
- `https://github.com/org/repo.git#subdir`

Template cache root defaults to `~/.boilersync/templates` (or `BOILERSYNC_TEMPLATE_DIR`).
GitHub is the only supported host for URL references.

## Project Tracking

BoilerSync writes `.boilersync` metadata in project roots after scaffold/pull so future operations can resolve template provenance.
Canonical `.boilersync` shape:

```json
{
  "template": "https://github.com/org/repo.git#subdir",
  "name_snake": "project_name",
  "name_pretty": "Project Name",
  "variables": {},
  "children": []
}
```

`source` metadata is no longer used.

## Template Authoring (Canonical)

# Boilersync Templates Directory

This directory contains project templates for generating new codebases with boilersync.

## Creating a New Template

To create a new boilersync template, create a template directory here with your template files.  There is no required structure, but the files in the template directory can contain interpolation slots using the template syntax explained in the following section. 

```
./my-template/
├── ... (template files) ...
├── README.starter.md
└── template.json          # Required template metadata
```

## Template Syntax

Use `$${variable_name}` for variable substitution in file contents:

```python
from $${name_snake}.cli import main

class $${name_pascal}Config(AppConfig):
    """$${name_pretty} - $${description}"""
```

Common variables:

- `$${name_snake}`, `$${name_kebab}`, `$${name_pascal}`, `$${name_pretty}` - Project name in various formats
- `$${author_name}`, `$${author_email}`, `$${author_github_name}` - Author info
- `$${description}` - Project description
- `$${python_version}` - Python version requirement

Browse existing templates for other available variables before adding a new one - an existing template may already include the variable you are looking for.  If it does not exist already, you may simply add a new one by using it; boilersync will adapt, and there is no need to declare it elsewhere.

## Variables in File and Folder Names

To use a variable in a filename or directory name, convert it to ALL CAPS:

| Variable | Filename Form |
|----------|---------------|
| `$${name_snake}` | `NAME_SNAKE` |
| `$${name_kebab}` | `NAME_KEBAB` |

Example:
```
/cli/NAME_SNAKE/__init__.py  →  /my_project/__init__.py
```

## File Extensions

### `.starter`

Template files with `.starter` in the extension are included in the initial boilersync generation but are **not kept in sync** with the template afterwards. Use this for files the user is expected to modify:

- `cli.starter.py` → `cli.py` (generated once, then user-owned)
- `README.starter.md` → `README.md` (user is expected to add more details to the README later)

Files **without** `.starter` remain linked to the template. When you run `boilersync push`, modifications to these files can update the template.

### `.boilersync`

Adding `.boilersync` to any filename prevents auto-formatters from corrupting template syntax. The extension is stripped during processing:

- `pyproject.toml.boilersync` → `pyproject.toml`

## Template Inheritance

Use `template.json` to extend another template:

```json
{
  "extends": "pip-package"
}
```

Child templates inherit all files from the parent.

## Runtime Metadata (`template.json`)

In addition to inheritance, `template.json` can define defaults and runtime behavior:

- `defaults`: default values for interpolation variables, applied before missing-variable collection
- `children`: initialize child templates during `boilersync init`
- `hooks`: run shell commands with `pre_init` and `post_init`
- `github`: optional GitHub repository creation settings
- `skip_git`: skip local git initialization when true

Template defaults may be literal values or `$${...}` / Jinja-rendered strings. Existing values win, so explicit `--var` values, saved project metadata, and earlier inherited-template defaults are not overwritten. Defaults are saved into generated project `.boilersync` metadata after init or pull.

Useful default patterns:

```json
{
  "defaults": {
    "api_package_name": "$${name_snake}_api",
    "django_app_name": "$${name_snake}",
    "web_package_name": "$${name_kebab}-web",
    "api_client_package_name": "$${name_kebab}-api-client",
    "api_client_export_name": "$${name_camel}",
    "cdn_base_url": "https://cdn.openbase.app/$${name_kebab}/",
    "with_frontend": true
  }
}
```

BoilerSync infers the default project name from the target folder. A trailing `-workspace` / `_workspace` suffix is stripped, so `woo-score-workspace` defaults to `name_snake=woo_score`. Override with `--var name_snake=...` when needed.

If a template references `github_user` and the value is not supplied, BoilerSync tries `gh api user --jq .login`. If the GitHub CLI is unavailable or unauthenticated, `github_user` remains a required variable and interactive prompting or `--var github_user=...` is still needed.

Hook step fields:

- `id` (optional): log identifier
- `run` (required): shell command to execute
- `condition` (optional): boolean expression to gate execution
- `cwd` (optional): working directory relative to target directory
- `env` (optional): env var map; values support `$${...}` interpolation
- `allow_failure` (optional): continue on non-zero exit code when true

Example:

```json
{
  "hooks": {
    "pre_init": [
      {
        "id": "deps",
        "run": "uv sync"
      }
    ],
    "post_init": [
      {
        "id": "format",
        "run": "uv run ruff format .",
        "allow_failure": true
      }
    ]
  }
}
```

### Block Overrides

Templates support Jinja2-style blocks for composable inheritance:

```toml
$${% block scripts %}
$${ super() }
$${cli_command} = "$${name_snake}.cli:main"
$${% endblock %}
```

- `$${% block name %}...$${% endblock %}` - Define overridable blocks in the parent template.
- `$${ super() }` - Include parent block content
- `$${% if condition %}...$${% endif %}` - Conditional content
