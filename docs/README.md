# AEnvironment Documentation

This directory contains the documentation for AEnvironment built with
[Jupyter Book 1](https://jupyterbook.org/).

## Building the Documentation

### Prerequisites

Install the required dependencies:

```bash
uv sync --project aenv --extra docs
```

### Steps

1. Build the documentation:

   ```bash
   cd docs
   jupyter-book build . --all
   ```

2. Preview the documentation:

   ```bash
   open _build/html/index.html
   ```

## Documentation Structure

The documentation content lives under `docs/` and is organized into sections that map
to the Jupyter Book table of contents in `_toc.yml`.

### Content directories

- **`getting_started/`**
  Installation and first-run materials.
- **`guide/`**
  User-facing guides for environments, tools, SDK, CLI, and MCP integration.
- **`architecture/`**
  System design and component-level architecture documents.
- **`examples/`**
  End-to-end examples and integrations.
- **`benchmarks/`**
  Benchmark methodology and results.
- **`development/`**
  Contributor-focused docs (building, contributing, workflows).

### Key files

- **`_toc.yml`**
  Authoritative navigation structure for the book.
- **`_config.yml`**
  Jupyter Book/Sphinx configuration.
- **`intro.md`**
  Book landing page (configured as `root` in `_toc.yml`).

### Recommended reading order

1. `getting_started/`
2. `guide/`
3. `examples/`
4. `architecture/` (as needed)
5. `development/` (if you plan to contribute)

### Adding a new documentation page

1. Create a new Markdown file under the appropriate folder (for example, `guide/your_topic.md`).
2. Add the page to `_toc.yml` so it shows up in the rendered navigation.

   Example (add a new chapter under the `User Guide` part):

   ```yaml
   - caption: User Guide
     chapters:
       - file: guide/environments
       - file: guide/tools
       - file: guide/your_topic
   ```

   Note that `file:` paths are relative to `docs/`, and you should omit the `.md` extension.
3. Build and preview to verify the page renders correctly:

   ```bash
   cd docs
   jupyter-book build . --all
   open _build/html/index.html
   ```
