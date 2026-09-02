# Minder docs

The documentation site for [Minder](https://github.com/minderhq/minder), built
with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

```bash
pip install -r requirements.txt
mkdocs serve       # live preview at http://127.0.0.1:8000
mkdocs build --strict
```

Content lives in `docs/`. The nav is defined in `mkdocs.yml`. CI builds `--strict`
on every PR and deploys `main` to GitHub Pages.

> Currently covers the **plugin ecosystem** in full. Self-hosting / architecture /
> operations pages are being verified against the code before they're published
> here (see [minderhq/minder#1248](https://github.com/minderhq/minder/issues/1248)).
