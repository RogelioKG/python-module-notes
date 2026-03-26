# ruff

[![RogelioKG/ruff](https://img.shields.io/badge/Sync%20with%20HackMD-grey?logo=markdown)](https://hackmd.io/@RogelioKG/ruff)

## Brief
定位如 ESLint，集 Linter、Formatter 於一身。

## References
+ 🔗 [**Python 開發：Ruff Linter、Formatter 介紹 + 設定教學**](https://blog.kyomind.tw/ruff/)

## Config

+ `.vscode/settings.json`
  ```json
  {
    "[python]": {
      "editor.formatOnSave": true,
      "editor.codeActionsOnSave": {
        "source.fixAll.ruff": "explicit",
        "source.organizeImports": "explicit"
      },
      "editor.defaultFormatter": "charliermarsh.ruff"
    },
    "python.analysis.typeCheckingMode": "basic"
  }
  ```

+ `pyproject.toml`
  ```toml
  [tool.ruff.lint]
  select = [
    "E",  # pycodestyle errors
    "W",  # pycodestyle warnings
    "F",  # pyflakes
    "I",  # isort
    "C",  # flake8-comprehensions
    "B",  # flake8-bugbear
    "UP", # pyupgrade
  ]

  [tool.ruff.format]
  quote-style = "double"
  ```

## Usage

### `check` Linter 檢查

  > `--fix` Linter 修正

  ```bash
  ruff check                          # Lint all files in the current directory (and any subdirectories).
  ruff check path/to/code/            # Lint all files in `/path/to/code` (and any subdirectories).
  ruff check path/to/code/*.py        # Lint all `.py` files in `/path/to/code`.
  ruff check path/to/code/to/file.py  # Lint `file.py`.
  ruff check @arguments.txt           # Lint using an input file, treating its contents as newline-delimited command-line arguments.
  ```

### `format` Formatter 修正
  
  > `--check` Formatter 檢查