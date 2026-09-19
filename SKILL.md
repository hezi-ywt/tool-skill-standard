---
name: tool-skill-standard
description: Design or review a reusable tool project that bundles a Python package, CLI scripts, tests, and one or more Agent Skills. Use when creating a decoupled capability for multiple projects, deciding package/Skill boundaries, defining project-local installation, or reviewing a tool-and-pipeline architecture.
---

# Reusable tool + Skill standard

Use this Skill when a capability should be reusable across projects while each
consumer builds its own pipeline. Keep the shared implementation independent
from any consumer's cards, prompts, batch names, output paths, machine paths,
or business rules.

## Standard project shape

```text
<tool-project>/
|-- pyproject.toml
|-- README.md
|-- src/
|   `-- <package-name>/       # reusable core package
|-- scripts/                  # project install/build/maintenance tools
|-- tests/                    # package tests and request-construction tests
`-- skills/
    `-- <skill-name>/
        |-- SKILL.md
        |-- references/       # on-demand Skill material
        `-- scripts/          # Skill-specific thin entry points
```

`src/` is the only home for reusable business logic. Root `scripts/` handles
project operations. A Skill's `scripts/` contains only thin wrappers and
diagnostic helpers; it must not become a second implementation of the package.

## Layering

```text
Agent
  -> Skill: discovery, configuration, operation boundaries, review gates
Consumer project pipeline: domain defaults, batching, prompts, outputs
  ->
Shared package: types, protocols, requests, responses, errors
  ->
External service or local runtime
```

The shared package exposes stable protocols and typed operations. A consumer
project adapts its domain objects to those protocols and owns orchestration,
retry policy, archival, and project metadata. Do not import a consumer project
from the shared package.

## Installation boundary

Project-local installation is the default. A consumer creates or selects its
own virtual environment and installs the shared package there:

```powershell
python -m pip install -e <tool-project>
```

The tool repository's `.venv` is only for developing and testing the tool.
Do not use it as a consumer runtime. Do not modify global Python or the global
Skill directory unless the user explicitly requests a global installation.

If the tool needs an installer, `scripts/install.py` must derive the project
root from its own file, create a project-local `.venv` by default, install the
package there, and support an explicit `--venv` override. It must not silently
copy code or Skills into global locations.

## Skill contract

Each `skills/<skill-name>/SKILL.md` must state:

1. what capability triggers it;
2. how the consumer project installs the package in its own environment;
3. how the service URL, port, or other configuration is supplied;
4. which operations are read-only and which have external side effects;
5. the package's main entry points and input protocols;
6. the preflight, tests, and completion criteria.

Keep long schemas and branch-specific procedures in that Skill's
`references/`, and keep Skill-specific helpers in its `scripts/`. Use relative
paths, environment variables, or placeholders. Never hard-code drive letters,
user directories, private addresses, credentials, or a consumer project name.

## Configuration and safety

Use this precedence unless the consumer project has a documented reason to
change it:

```text
CLI arguments > AUTOMUSE_* / tool-specific environment variables
> consumer project's local configuration > safe defaults
```

Read-only preflight must not generate, upload, delete, restart, clear queues,
or mutate service settings. Real execution and recovery commands must state
their side effects. Credentials and tokens must stay out of logs, Skills,
README files, and output metadata.

## Validation gate

Before delivering a new tool project:

- import the package from `src/`;
- run CLI `--help` and read-only preflight checks;
- test URL, timeout, authentication, payload construction, success responses,
  errors, and timeouts offline;
- test output files and metadata sidecars;
- install the package into a separate consumer project's environment and make
  one representative call or dry-run;
- validate every bundled Skill and verify its relative references.

Report the package version, installation scope, tests, external connections,
and any real side effects. Keep the shared package generic; put project
pipeline behavior in the consuming project.
