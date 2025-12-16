# Contributing Guide

Thank you for your interest in contributing to AEnvironment! This guide will help you get started.

## Code of Conduct

Please read and follow our [Code of Conduct](https://github.com/inclusionAI/AEnvironment/blob/main/CODE_OF_CONDUCT.md).

## Getting Started

### Prerequisites

- Python 3.12+
- Go 1.21+
- Docker
- Kubernetes (optional, for full platform testing)

### Development Setup

1. **Clone the repository**

```bash
git clone https://github.com/inclusionAI/AEnvironment.git
cd AEnvironment
```

1. **Set up Python SDK**

```bash
cd aenv
pip install -e ".[dev]"
```

1. **Set up Go components**

```bash
cd controller
go mod download
```

1. **Install pre-commit hooks**

```bash
pip install pre-commit
pre-commit install
```

## Project Structure

```bash
AEnvironment/
├── aenv/                 # Python SDK
│   ├── src/
│   │   ├── aenv/        # Core SDK
│   │   └── cli/         # CLI tool
│   └── examples/        # Usage examples
├── api-service/         # API gateway (Go)
├── controller/          # Environment controller (Go)
├── envhub/              # Environment registry (Go)
├── deploy/              # Helm charts
└── docs/                # Documentation
```

## Development Workflow

### 1. Create a Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/your-bug-fix
```

### 2. Make Changes

Follow the coding standards for each component:

- **Python**: PEP 8, type hints, docstrings
- **Go**: Go conventions, golangci-lint

### 3. Write Tests

All changes should include tests:

```bash
# Python tests
cd aenv
pytest tests/ -v

# Go tests
cd controller
go test ./...
```

### 4. Run Linters

```bash
# Python
cd aenv
black src/
isort src/
mypy src/
ruff check src/

# Go
cd controller
golangci-lint run
```

### 5. Commit Changes

Follow conventional commits:

```bash
git commit -m "feat: add new tool registration API"
git commit -m "fix: resolve environment cleanup issue"
git commit -m "docs: update SDK reference"
```

### 6. Submit Pull Request

- Push your branch
- Create a PR against `main`
- Fill out the PR template
- Wait for review

## Coding Standards

### Python

```python
"""Module docstring describing the module."""

from typing import Optional, List

from aenv.core.exceptions import ToolError


def my_function(
    param1: str,
    param2: int = 42,
    param3: Optional[List[str]] = None
) -> dict:
    """Short description of function.

    Longer description if needed.

    Args:
        param1: Description of param1
        param2: Description of param2
        param3: Description of param3

    Returns:
        Description of return value

    Raises:
        ToolError: When something goes wrong

    Example:
        >>> result = my_function("test")
        >>> print(result)
        {"status": "ok"}
    """
    if param3 is None:
        param3 = []

    return {"status": "ok"}
```

### Go

```go
// Package controller provides environment lifecycle management.
package controller

import (
    "context"
    "fmt"
)

// MyFunction does something useful.
//
// It takes a context and configuration, returning a result or error.
func MyFunction(ctx context.Context, config *Config) (*Result, error) {
    if config == nil {
        return nil, fmt.Errorf("config is required")
    }

    // Implementation
    return &Result{}, nil
}
```

## Testing

### Unit Tests

```python
# tests/test_environment.py
import pytest
from aenv import Environment

@pytest.mark.asyncio
async def test_environment_creation():
    env = Environment("test-env")
    assert env.name == "test-env"

@pytest.mark.asyncio
async def test_tool_execution():
    async with Environment("test-env") as env:
        result = await env.call_tool("echo", {"message": "hello"})
        assert not result.is_error
```

### Integration Tests

```python
# tests/integration/test_full_workflow.py
import pytest
from aenv import Environment

@pytest.mark.integration
@pytest.mark.asyncio
async def test_full_workflow():
    async with Environment("integration-test-env") as env:
        # Test complete workflow
        tools = await env.list_tools()
        assert len(tools) > 0

        result = await env.call_tool(tools[0].name, {})
        assert not result.is_error
```

### Running Tests

```bash
# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=aenv --cov-report=html

# Run specific test file
pytest tests/test_environment.py -v

# Run integration tests
pytest tests/integration/ -v -m integration
```

## Documentation

### Building Docs

```bash
cd docs
pip install jupyter-book
jupyter-book build . --all
```

### Writing Docs

- Use Markdown with MyST extensions
- Include code examples
- Add diagrams with Mermaid
- Keep examples runnable

## Pull Request Guidelines

### PR Title

Use conventional commit format:

- `feat: Add new feature`
- `fix: Fix bug in X`
- `docs: Update documentation`
- `refactor: Refactor code`
- `test: Add tests`
- `chore: Update dependencies`

### PR Description

Include:

1. **What**: Brief description of changes
1. **Why**: Motivation for the change
1. **How**: Implementation approach
1. **Testing**: How it was tested
1. **Screenshots**: If UI changes

### PR Checklist

- [ ] Tests pass locally
- [ ] Linters pass
- [ ] Documentation updated
- [ ] Changelog updated (if applicable)
- [ ] No breaking changes (or documented)

## Review Process

1. **Automated checks**: CI runs tests and linters
1. **Code review**: At least one maintainer review
1. **Approval**: Maintainer approves
1. **Merge**: Squash and merge

## Release Process

### Versioning

We use semantic versioning:

- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes

### Creating a Release

1. Update version in `pyproject.toml` / `go.mod`
1. Update CHANGELOG.md
1. Create release PR
1. After merge, tag the release
1. CI builds and publishes

## Getting Help

- **GitHub Issues**: Bug reports and feature requests
- **Discussions**: Questions and ideas
- **Slack**: Real-time chat (link in README)

## Recognition

Contributors are recognized in:

- CONTRIBUTORS.md
- Release notes
- Project documentation

Thank you for contributing to AEnvironment! 🎉
