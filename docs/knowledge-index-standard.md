# Knowledge Index Standard (Master Repo)

Date: 2026-07-21

Purpose:
Keep infra-docs as the master knowledge index for QUIRCH FOODS repositories without duplicating operational content from source repos.

## Core rule

infra-docs is an index and pointer hub.
Do not copy full runbooks, procedures, or scripts from domain repositories when a canonical source already exists.

## Canonical-source model

For each topic tracked in infra-docs:

- Record the owning repository.
- Record canonical links to the exact source files.
- Keep only a short local pointer summary.
- Store operational detail in the owning repo.

## Required fields for pointer records

Every pointer record in docs/ should include:

- Topic title
- Owning repo URL
- Canonical file links (guide/script/config)
- Last verified date
- One-line usage intent (what to open first)

## Update workflow

1. Update content in the owning repo first.
2. Validate links still resolve.
3. Update the pointer record in infra-docs (links + last verified date only).
4. Do not paste duplicated step-by-step content into infra-docs.

## Allowed exceptions

You may keep local content in infra-docs when it is:

- Cross-repo context that does not belong to a single repo.
- Environment-specific gotchas discovered during operations.
- Decision logs and architecture summaries.

Do not use exceptions to duplicate full source runbooks.

## Quick template

Use this structure for new pointer records:

- Topic: <name>
- Owning repo: <repo URL>
- Canonical guide: <file URL>
- Canonical script/config: <file URL>
- Last verified: YYYY-MM-DD
- Notes: short pointer only
