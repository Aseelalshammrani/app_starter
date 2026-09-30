# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A Python package of document-related tools (converting and processing document formats) exposed to AI assistants through an MCP server built on `FastMCP` from `mcp==1.8.0`. Dependencies are managed with `uv` (see `pyproject.toml` / `uv.lock`); `markitdown[docx,pdf]` does the document conversion.

## Commands

The dev environment lives in `.venv`. It does not need to be activated: always prefix commands with `uv run` so they use the project environment instead of the system Python.

```bash
uv venv && uv pip install -e .                     # first-time setup
uv run main.py                                     # start the MCP server (stdio; blocks until stopped)
uv run pytest                                      # all tests
uv run pytest tests/test_document.py               # one file
uv run pytest tests/test_document.py::TestBinaryDocumentToMarkdown::test_binary_document_to_markdown_with_pdf   # one test
```

No linter or formatter is configured.

## Conventions

- Always apply appropriate type hints to function arguments (and return values), in every function, not only MCP tools.

## Architecture

- `main.py` is the server entry point. It creates `mcp = FastMCP("docs")`, imports tool functions from the `tools` package and registers each one with `mcp.tool()(func)`. A function in `tools/` is **not** exposed to MCP clients until it is registered here.
- `tools/` holds plain Python functions, one module per domain:
  - `tools/math.py`: `add`, the reference example of a fully documented tool (it is the only one currently registered).
  - `tools/document.py`: `binary_document_to_markdown(binary_data, file_type)`, which wraps the bytes in `BytesIO` and converts them with `MarkItDown` using `StreamInfo(extension=file_type)` (e.g. `"docx"`, `"pdf"`). It is not yet registered in `main.py` and does not yet follow the tool conventions below.
- `tests/` tests the tool functions directly (not through the MCP server), using binary fixtures in `tests/fixtures/` (`mcp_docs.docx`, `mcp_docs.pdf`).

## Defining MCP tools

Tools are Python functions registered with the server:

```python
mcp.tool()(my_function)
```

FastMCP builds the tool schema shown to the AI from the function signature, type hints, `Field` descriptions and docstring, so these are part of the tool's interface, not just documentation.

**Parameters:** describe every parameter with pydantic's `Field` and give it a type hint and a return type:

```python
from pydantic import Field

def my_tool(
    param1: str = Field(description="Detailed description of this parameter"),
    param2: int = Field(description="Explain what this parameter does"),
) -> ReturnType:
    """Comprehensive docstring here"""
    # Implementation
```

**Docstring:** the tool description should:

1. Begin with a one-line summary.
2. Give a detailed explanation of what the tool does.
3. Explain when to use (and when not to use) the tool.
4. Include usage examples with expected input/output.

`tools/math.py` (`add`) is the model to copy:

```python
def add(
    a: float = Field(description="First number to add"),
    b: float = Field(description="Second number to add"),
) -> float:
    """Add two numbers together.

    Takes two numerical inputs and returns their sum. This tool handles
    integers and floating point numbers.

    When to use:
    - When you need to perform simple addition
    - When you need precise numerical calculation

    Examples:
    >>> add(2, 3)
    5.0
    >>> add(2.5, 3.5)
    6.0
    """
    return a + b
```

**Adding a tool:** write the function in the appropriate `tools/` module following the conventions above, import it in `main.py` and register it with `mcp.tool()(...)`, then add tests under `tests/` (put any sample files in `tests/fixtures/`).
