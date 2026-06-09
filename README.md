# Ontology-template

Template for ontologies. Structured in view of using shared **continuous integration** and **documentation** workflows.

Currently, the following workflows are pre-defined:

- continuous integration (CI) (`.github/workflows/ci.yml`): formal checks (ROBOT, reasoning, SHACL...)
- WIDOCO generation (`.github/workflows/widoco.yml`): generates ontology HTML docs under `docs/`
- Wiki extraction (`.github/workflows/wiki-extract.yml`): exports GitHub Wiki into `docs/from-wiki/` (md/tex/pdf)

The shared workflows are in GitHub repository Airy59/OntoQA.

## Configuration

The OntoQA reusable workflows expect an `ontology/src/config.md` file (YAML front matter) that specifies:

- `main_ontology`: ontology entrypoint file (relative path)
- `reasoner`: e.g. `hermit`, `pellet`, `elk`, `jfact`, `structural`

This template ships a placeholder file at `ontology/src/config.md` that you must update.

## Running workflows

All workflows in this template are configured for **manual runs only** (Actions tab).

## Wiki documentation prerequisites

To generate wiki documentation, GitHub Wiki must be:

- enabled in repository settings, and
- initialized with at least one page.

See OntoQA’s setup guide: `https://raw.githubusercontent.com/Airy59/OntoQA/main/docs/WIKI_SETUP.md`.

Replace this file with a proper readme.md describing your project.

Replace the default CC BY-SA 4.0 license (that applies to the present template) by the license suitable for your project.
