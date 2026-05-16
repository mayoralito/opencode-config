# Installation Guide

Provide these instructions to the user. Do not run them yourself.

## Install uv (required)

**macOS/Linux (recommended):**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Using pip:**

```bash
pip install uv
```

**Using pipx:**

```bash
pipx install uv
```

**Using Homebrew (macOS):**

```bash
brew install uv
```

After installing uv, the user may need to restart their terminal or run:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## Verify installation

```bash
which uv
```

No need to install cookiecutter separately — `uvx cookiecutter` handles it automatically.
