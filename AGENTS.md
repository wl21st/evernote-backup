# AGENTS.md

This file contains guidelines and commands for agentic coding agents working on this evernote-backup repository.

## Development Environment Setup

This project uses Poetry for dependency management and packaging.

```bash
# Install dependencies
poetry install

# Install with test dependencies
poetry install --with test,dev
```

## Build / Lint / Test Commands

### Testing Commands
```bash
# Run all tests
poetry run pytest

# Run tests with coverage
poetry run pytest --cov=evernote_backup tests/

# Run a single test file
poetry run pytest tests/test_note_storage.py

# Run a specific test function
poetry run pytest tests/test_note_storage.py::test_database_file_missing

# Run tests with verbose output
poetry run pytest -v

# Run tests with coverage and XML report (used in CI)
poetry run pytest --cov-report=xml --cov=evernote_backup tests/
```

### Linting and Formatting Commands
```bash
# Format code with ruff
poetry run ruff format .

# Lint code with ruff
poetry run ruff check .

# Format and lint in one command
poetry run ruff format . && poetry run ruff check .

# Run pre-commit hooks manually
poetry run pre-commit run --all-files
```

### Type Checking
```bash
# Run mypy type checking
poetry run mypy evernote_backup
```

### Build Commands
```bash
# Build the package
poetry build

# Install the built package in development mode
poetry install --no-dev
```

## Code Style Guidelines

### General Principles
- **Python Version**: Target Python 3.9+ (as specified in pyproject.toml)
- **Code Formatting**: Use ruff format (configured to match Black style)
- **Line Length**: 88 characters (ruff.format.line-ending = "lf")
- **Imports**: Use isort-style import sorting with Black profile
- **Type Hints**: Required for all new functions (mypy enforces this)

### Import Organization
```python
# Standard library imports first
import logging
import sqlite3
from pathlib import Path
from typing import NamedTuple, Optional

# Third-party imports next
import click
from evernote.edam.type.ttypes import Note

# Local imports last
from evernote_backup.config import CURRENT_DB_VERSION
from evernote_backup.log_util import get_logger
```

### Type Hints and Function Signatures
- All functions must have type hints for parameters and return values
- Use `Optional[T]` for nullable types
- Use `Union[T, U]` for multiple possible types
- Use `NamedTuple` for simple data structures
- Use `Path` instead of strings for file paths

```python
from typing import Optional, NamedTuple
from pathlib import Path

class NoteForSync(NamedTuple):
    guid: str
    title: str
    linked_notebook_guid: Optional[str]

def process_note(note_path: Path, user_id: Optional[str] = None) -> bool:
    """Process a note file and return success status."""
    pass
```

### Naming Conventions
- **Variables and functions**: `snake_case`
- **Classes**: `PascalCase` 
- **Constants**: `UPPER_SNAKE_CASE` (but note WPS115 is ignored)
- **Private methods**: `_underscore_prefix`
- **Modules**: `snake_case.py`

### Error Handling
- Use specific exception types rather than generic `Exception`
- Log errors using the logging module
- Use custom exceptions like `ProgramTerminatedError` for CLI termination
- Handle network errors with retry logic

```python
import logging

logger = logging.getLogger(__name__)

def risky_operation():
    try:
        # Some operation that might fail
        pass
    except NetworkError as e:
        logger.error(f"Network operation failed: {e}")
        raise
    except SpecificError as e:
        logger.warning(f"Expected error occurred: {e}")
        return False
```

### Logging Guidelines
- Use module-level loggers: `logger = logging.getLogger(__name__)`
- Use appropriate log levels: DEBUG, INFO, WARNING, ERROR, CRITICAL
- For CLI output, use click.echo() or similar, not print()
- Use `log_format_note()` and `log_format_notebook()` utilities for consistent formatting

### Database and Storage
- Use SQLite with the `SqliteStorage` class from `note_storage.py`
- Store complex objects as pickled/lzma-compressed BLOB data
- Use context managers for database connections
- Follow the existing schema patterns in `DB_SCHEMA`

### CLI Development
- Use Click for CLI interfaces (see `cli.py`)
- Group related options using `click_option_group`
- Use the `@handle_errors` decorator for consistent error handling
- Follow the existing parameter naming patterns

### Testing Guidelines
- Write tests in the `tests/` directory
- Use pytest with fixtures defined in `conftest.py`
- Use `tmp_path` fixture for temporary files/directories
- Mock external dependencies using `pytest-mock`
- Test both success and error paths

```python
import pytest
from pathlib import Path

def test_function_with_temp_file(tmp_path):
    test_file = tmp_path / "test.txt"
    result = function_to_test(test_file)
    assert result.success
    assert test_file.exists()
```

### Code Organization
- Keep modules focused on single responsibilities
- Use utility modules for shared functionality (e.g., `log_util.py`, `cli_app_util.py`)
- Separate CLI logic from business logic
- Use the existing package structure under `evernote_backup/`

### Security Considerations
- Never log sensitive data (tokens, passwords)
- Use secure defaults for SSL connections
- Validate input parameters, especially from CLI
- Be careful with unpickling data from untrusted sources

### Performance Guidelines
- Use caching for expensive operations
- Limit memory usage for large data processing
- Use streaming for large file operations
- Consider rate limiting for API calls

## Project Structure

```
evernote_backup/
├── cli.py                 # Main CLI entry point
├── cli_app*.py           # CLI application logic
├── note_*.py             # Note processing modules
├── evernote_client*.py   # Evernote API client
├── storage.py            # Database operations
├── log_util.py           # Logging utilities
├── config*.py            # Configuration
└── version.py            # Version info

tests/
├── conftest.py           # pytest fixtures
├── test_*.py             # Test files
└── __init__.py
```

## Common Patterns

### Error Handling Decorator
Use `@handle_errors` from `cli.py` for CLI commands to ensure consistent error reporting.

### Logging Setup
Use `init_logging()` from `log_util.py` to configure logging based on verbosity flags.

### Path Handling
Always use `Path` objects from `pathlib` for file operations. The project includes type helpers like `FILE_ONLY` and `DIR_ONLY` for Click options.

### Configuration
Default configuration values are in `config_defaults.py`. Use these rather than hardcoded values.

## Notes

- The project supports both Evernote and Yinxiang (China) backends
- OAuth and password-based authentication are supported
- The database uses SQLite with a custom schema for notes, notebooks, tasks, and reminders
- Export functionality generates ENEX files compatible with Evernote
- The codebase follows wemake-python-styleguide guidelines with some rules ignored