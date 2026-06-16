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
- **After moving templates, scan each moved `.html` file for `{% extends %}` and
  `{% include %}` tags. Add the `app_name/` prefix to every template name
  referenced in those tags (e.g. `{% extends "app_name/base.html" %}`).**

## Step 3 — Fix imports

After all moves:
1. Scan every `.py` file for broken imports and update each one
2. Fix view references in both project-level and app-level `urls.py`
3. Fix `INSTALLED_APPS` and any hardcoded paths in `settings.py`
4. Ensure `TEMPLATES[0]['DIRS']` points to `templates/` at project root
5. Ensure `STATICFILES_DIRS` points to the root-level `static/` directory
   (not `app_name/static/`)

## Step 4 — Report

Produce a summary:
- Every moved file: old path → new path
- Every fixed import: old line → new line
- Every template `{% extends %}` / `{% include %}` fix: old → new
- Flagged uncertain files with explanation

**Do NOT run the server or execute migrations.**

## Constraints

- Do not rename any class, function, or variable
- Do not modify business logic — only file locations and import statements
- Preserve all comments and docstrings exactly as-is
- Create `__init__.py` wherever needed
