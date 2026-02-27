# BoilerSync Essentials

This document is a compact reference for the core BoilerSync workflow plus canonical template-authoring guidance.

## Core Workflow

```bash
# Clone template source repo into local cache
boilersync templates init https://github.com/your-org/your-templates.git

# Initialize a project from a source-qualified template
boilersync init your-org/your-templates#python/service-template

# Pull template updates later
boilersync pull

# Push committed project changes back to template source
boilersync push
```

## Command Essentials

- `boilersync init TEMPLATE_REF`: create a project from a template (requires empty target directory)
- `boilersync pull [TEMPLATE_REF]`: apply template updates to an existing project
- `boilersync push`: copy committed project improvements back to template source
- `boilersync templates init`: clone and register a template source repo in local cache

Use `boilersync --help` and subcommand `--help` for flags.

## Template Reference Formats

- `org/repo#subdir`
- `https://host/org/repo.git#subdir`
- `git@host:org/repo.git#subdir`
- legacy local names (deprecated)

Default template cache:

```bash
~/.boilersync/templates
```

Override with:

```bash
BOILERSYNC_TEMPLATE_DIR=/custom/path
```

## Project Metadata

After `init` and `pull`, BoilerSync writes `.boilersync` metadata to project roots so future operations can resolve template provenance.

## Template Authoring (Canonical)

The section below is copied from:
`/Users/gabemontague/.boilersync/templates/openbase-community/templates/CLAUDE.md`

# Boilersync Templates Directory

This directory contains project templates for generating new codebases with boilersync.

## Creating a New Template

To create a new boilersync template, create a template directory here with your template files.  There is no required structure, but the files in the template directory can contain interpolation slots using the template syntax explained in the following section. 

```
./my-template/
├── ... (template files) ...
├── README.starter.md
└── template.json          # Optional, for inheritance
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
