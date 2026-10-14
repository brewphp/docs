# Contributing

Thank you for helping BrewPHP! The full guidelines (code style, tests,
commit messages, how pull requests are reviewed) are in the organisation's
[CONTRIBUTING.md](https://github.com/brewphp/.github/blob/main/CONTRIBUTING.md).

BrewPHP is split into several repositories:

| Repository | Contents |
|---|---|
| [brewphp/brewphp](https://github.com/brewphp/brewphp) | The framework |
| [brewphp/docs](https://github.com/brewphp/docs) | This documentation |
| [brewphp/skeleton](https://github.com/brewphp/skeleton) | The starter project |

Every new feature needs code, tests and a documentation page.

## Building the documentation locally

The documentation is written in Markdown and built with
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/). You need
Python 3.9 or newer.

```bash
git clone https://github.com/brewphp/docs.git
cd docs
python -m venv .venv
source .venv/bin/activate        # on Windows: .venv\Scripts\activate
pip install -r requirements.txt
mkdocs serve
```

Open <http://127.0.0.1:8000/>; pages reload as you edit them.

Before opening a pull request, check that the build has no warnings, exactly
as Read the Docs will run it:

```bash
mkdocs build --strict
```

## Writing style

- Write in plain, simple English, for readers who are new to BrewPHP.
- Every code block must be copy-paste runnable against the current release.
- Show commands together with their output.
- Use admonitions (`!!! tip`, `!!! warning`) for tips and pitfalls.
