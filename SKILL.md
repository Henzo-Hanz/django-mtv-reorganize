---
name: django-mtv-reorganize
description: >
  Reorganizes a Django project into strict MTV (Model–Template–View) architecture.
  Use this skill whenever the user asks to organize, restructure, clean up, or refactor
  a Django project's file layout — even if they say things like "my files are a mess",
  "templates are in the wrong place", "organize my Django app", or "follow Django conventions".
  Also triggers for single-app projects with a separate top-level utils/ module.
  Handles audit, file moves, import fixes, and settings verification in one pass.
---

# Django MTV Reorganizer

Reorganizes a Django project (single-app) into the canonical MTV structure plus a
top-level `utils/` module, then fixes all broken imports and settings references.

## Target structure

```
project_root/
├── manage.py
├── requirements.txt
├── .env
│
├── project_name/          ← Django config package
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
│
├── app_name/              ← Single Django app
│   ├── __init__.py
│   ├── apps.py
│   ├── admin.py
│   ├── models/            ← one file per model group
│   │   └── __init__.py    ← re-exports all models
│   ├── views/             ← one file per view group
│   │   └── __init__.py    ← re-exports all views
│   ├── forms/
│   │   └── __init__.py
│   ├── urls.py
│   └── migrations/
│       └── __init__.py
│
├── templates/
│   └── app_name/          ← all HTML files, namespaced
│
├── static/
│   ├── css/
│   ├── js/
│   └── img/
│
└── utils/                 ← top-level helpers, NOT inside the app
    ├── __init__.py
    └── *.py
```

## Step 1 — Audit (read-only)

Scan the entire codebase and produce a report:
- Every misplaced file → its target path
- Every file already correct → mark ✅
- Scan all HTML templates for `{% extends %}` and `{% include %}` tags — flag
  every cross-reference that will need updating (they'll need the `app_name/` prefix)
- Scan `.gitignore` — flag entries that reference any path that will cease to exist
  after the reorganization (e.g. `core/static/`, `core/templates/`, `app_name/security.py`)
- Flag if `staticfiles/` is tracked in git (it should be in `.gitignore` — it's a build
  artifact from `collectstatic`)
- Do NOT move anything yet

## Step 2 — Reorganize

Move files according to these rules:
- Contains only model classes → `app_name/models/`
- Contains only views / CBVs → `app_name/views/`
- Contains HTML → `templates/app_name/`
- Contains reusable helpers / adapters → `utils/`
- If `models/` or `views/` become packages, update their `__init__.py` to re-export
  everything so existing references keep working
- If unsure where a file belongs → move to `utils/` and flag it in Step 4
- Never delete any file
- **After each move batch, remove empty source directories and purge stale `__pycache__`.**
  Specifically:
  - After moving all templates out of `app_name/templates/`, delete `app_name/templates/`
  - After moving all static files out of `app_name/static/`, delete `app_name/static/` recursively
  - After moving any `.py` file, delete `__pycache__/` in both the **source** directory
    (where the file used to live) and the **destination** directory. Stale `.pyc` bytecache
    can cause Python to load old code silently (e.g. after splitting `models.py` into a
    `models/` package).
  - Only remove directories guaranteed empty after the move — never remove a directory
    that still has legitimate files (e.g. `migrations/`, `__init__.py`, `admin.py`, etc.)
  - Use `Get-ChildItem` to verify emptiness before removal
- **After moving templates, scan each moved `.html` file for `{% extends %}` and
  `{% include %}` tags. Add the `app_name/` prefix to every template name
  referenced in those tags (e.g. `{% extends "app_name/base.html" %}`).**

## Step 2.5 — Automatic `.gitignore` management

Edit `.gitignore` so it reflects the new file layout:

- **Add** `staticfiles/` if not present — the output of `collectstatic` is a build artifact
  and must not be versioned. (If it was already tracked, `git rm -r --cached staticfiles/`
  is needed — flag this in Step 4.)
- **Remove** any entry that referenced a path that no longer exists after the moves
  (e.g. `core/templates/`, `core/static/`, or any app-level file path that was moved
  to `utils/`). Stale entries are dead weight and make the file confusing.
- **Keep** standard entries: `__pycache__/`, `*.pyc`, `.env`, `db.sqlite3`, `.venv/`,
  `.coverage` — verify they are present; add any that are missing.
- **Never add** `templates/`, `static/`, or any source directory to `.gitignore` —
  those are project source files that should be committed.

**Do NOT track the new `templates/` or `static/` directories automatically** — leave
that to the user's discretion (`git add` is outside the skill's scope).

## Step 3 — Fix imports

After all moves:
1. Scan every `.py` file for broken imports and update each one.
   Pay special attention to files that moved packages:
   - `app_name/models/` → old `from .models import` must become `from core.models import`
   - `app_name/views/` → relative `from .security` imports must become absolute `from utils.security`
   - `utils/*.py` files that used `from .models import` must become `from core.models import`
2. Fix view references in both project-level and app-level `urls.py`
3. Fix `INSTALLED_APPS` and any hardcoded paths in `settings.py`
4. Ensure `TEMPLATES[0]['DIRS']` points to `templates/` at project root
5. Ensure `STATICFILES_DIRS` points to the root-level `static/` directory
   (not `app_name/static/`)
6. **Fix tests** — if `app_name/tests/` (directory package) exists:
   - Scan every `.py` inside it for broken imports (same rules as step 1)
   - If only `app_name/tests.py` exists, scan and fix that single file
   - Import paths like `from .security` or `from .models` changed; update them
7. **Validate** — run `python manage.py check` (or `./manage.py check`) to verify
   the project loads without import errors or misconfiguration. If it fails, inspect
   the error, fix the root cause, and re-run until clean.

## Step 4 — Report

Produce a summary:
- Every moved file: old path → new path
- Every fixed import: old line → new line
- Every template `{% extends %}` / `{% include %}` fix: old → new
- `.gitignore` changes:
  - Every entry added
  - Every entry removed
  - If `staticfiles/` was tracked and needs `git rm --cached`, flag it
- 🧹 Directories cleaned up (empty dirs removed, `__pycache__` purged)
- ✅ Result of `python manage.py check` (pass / fail details)
- Flagged uncertain files with explanation

**Do NOT run the server or execute migrations.**

## Constraints

- Do not rename any class, function, or variable
- Do not modify business logic — only file locations and import statements
- Preserve all comments and docstrings exactly as-is
- Never delete a file that contains logic (`.py`, `.html`, `.css`, `.js`, `.md`, configs, etc.)
  — **empty directories left by moves ARE exempt** and must be removed (Step 2).
- Create `__init__.py` wherever needed
