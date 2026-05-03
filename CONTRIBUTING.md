# Contributing to RedisSMQ Documentation

This repository holds the **language-agnostic** specification and conceptual documentation shared by both the 
TypeScript (`redis-smq`) and Go (`go-redis-smq`) implementations.

Since we do not use static site generators, complex CI pipelines, or version-switching JavaScript, we rely on a 
strict **manual folder-based versioning** system. Please read this guide carefully before opening a pull request.

---

## The Golden Rules

1. **All new features go exclusively into `/next/`.** Never edit frozen versioned folders (`/vX.Y/`) for feature work.
2. **No code examples.** This repository contains **conceptual documentation only**. Full code examples belong in the 
implementation repositories (`redis-smq` and `go-redis-smq`). Including them here creates double-maintenance and 
version-drift nightmares.

---

## Repository Structure (The Rules)

| Path                          | Content                                                                   | Rule                                                                                  |
|:------------------------------|:--------------------------------------------------------------------------|:--------------------------------------------------------------------------------------|
| **`/next/`**                  | Unreleased, bleeding‑edge docs for the `next` development branches.       | **Mutable.** This is the only place you should actively edit for new features.        |
| **`/vX.Y/`** (e.g., `/v1.0/`) | Frozen snapshots of specific stable releases.                             | **Immutable.** Do not edit these folders unless fixing a critical typo (see Patches). |
| **`/README.md`**              | Landing page with the Compatibility Matrix.                               | Update this on **every release** to add the new version row.                          |

---

## 1. Adding a New Feature (Daily Work)

When implementing a new feature (e.g., a new Queue type, a new exchange, or a configuration option):

1. **Always edit the files inside `/next/`**.
   - Update existing Markdown files or create new ones.
   - Describe the **behavior, data models, configuration schemas, Redis key structures, and flow diagrams** using plain English, Markdown tables, or Mermaid/ASCII diagrams.

2. **Do NOT add TypeScript or Go code examples.**
   - ❌ Bad: A full `const queue = new Queue(...)` or `queue := smq.NewQueue(...)` snippet.
   - ✅ Good: A JSON/YAML configuration block, a pseudo-code algorithm description, or a sequence diagram.
   - ✅ Good: A link pointing to the actual implementation repositories.

3. **Do NOT touch any versioned folders** (e.g., `/v10.3/`).

4. **Open a Pull Request** against the `main` branch of this repository.
   - **Title:** `docs: add <feature-name> to /next/`
   - **Description:** Mention which features are being added and link to the corresponding implementation PRs in `redis-smq` and `go-redis-smq`.

### Why No Code Examples?

The TypeScript and Go libraries version independently (e.g., `redis-smq` v10.3.0 vs `go-redis-smq` v1.2.0). If we 
freeze code examples inside `redis-smq-docs/v10.3/`, and later `go-redis-smq` introduces a breaking change in v2.0.0 
while TS stays on v10.3, the Go examples in the shared docs become **invalid and misleading**. Keeping code examples 
only in each implementation's own `examples/` folder guarantees they are always tested against that specific language's 
version.

### Blocking Rule (No Merge Without Code)

**You are responsible for ensuring that the Go and TypeScript implementations are ready for the docs change.**

- Before your docs PR is merged, the feature **must** be merged into both `redis-smq/next` **and** `go-redis-smq/next`.
- If the Go implementation is lagging, **do not merge the docs PR**. Keep it in draft until both sides are ready. This prevents `/next/` from advertising features that do not exist in both runtimes.

---

## 2. Cutting a New Stable Release (The Release Day)

When both `redis-smq` (TypeScript) and `go-redis-smq` (Go) have tagged a stable release 
(e.g., TS `v11.0.0` and Go `v1.0.0`), you must freeze the `next` documentation into a versioned folder.

Follow these steps **exactly** in your terminal:

```bash
# 1. Ensure you are on the main branch and up to date
git checkout main
git pull origin main

# 2. Freeze the volatile 'next' into a permanent snapshot
cp -R ./next ./v1.0

# 3. Update the Compatibility Matrix in the root README.md
#    Open README.md in your editor.
#    Add a new row to the table:
#    | /v1.0/ | >=11.0.0 <11.2.0 | >=1.0.0 <1.1.0 | ✅ Latest Stable |

# 4. Wipe the '/next/' folder to start a fresh development cycle
rm -rf ./next/*
echo "# ⚠️ Unreleased Development Docs" > ./next/README.md
echo "" >> ./next/README.md
echo "These docs map to redis-smq@next and go-redis-smq@next." >> ./next/README.md
echo "" >> ./next/README.md
echo "They are subject to change and may be incomplete." >> ./next/README.md

# 5. Commit everything
git add .
git commit -m "Release docs v1.0 (maps to redis-smq v11.0.0 and go-redis-smq v1.0.0)"

# 6. Tag the commit to easily find this release later
git tag v1.0

# 7. Push to GitHub
git push origin main --tags
```

### What about Major Versions? (e.g., `v12.0.0` vs `go-redis-smq v2.0.0`)

If the TypeScript spec jumps to `v12` and Go to `v2.0`, simply name the folder `v2.0` instead of `v1.1`. The update steps remain identical:

```bash
cp -R ./next ./v2.0
# Update README.md: v2.0 | >=12.0.0 | >=2.0.0
# ... etc ...
```

---

## 3. Patch Releases (Hotfixes)

If a critical bug is fixed in `redis-smq@v11.0.1` or `go-redis-smq@v1.0.1`:

- **Do NOT create a new folder** (e.g., `v1.0.1`). The folder `v1.0` already covers the entire `1.0.x` series.
- **Only edit the `README.md`** if the Compatibility Matrix needs to expand its range.
- **Only edit a specific file inside `/v1.0/`** if the patch changed the *public-facing behavior or configuration* in a way that must be reflected in the prose. This should be extremely rare.

---

## 4. Updating the Compatibility Matrix

The matrix in the root `README.md` is the most critical part of this repository. When adding a new version, always ensure the ranges are correct:

- **Use `>=` and `<`** to define the range (e.g., `>=10.3.0 <10.4.0`).
- **Patches are automatically covered** by the minor range. You do not need to update the matrix for `10.3.1`.

---

## 5. Commit Message Conventions

To maintain a clean and searchable project history, please follow the Conventional Commits specification.

```text
Format: <type>(<scope>): <description>
```

Examples:

- `docs(next): update API reference` — Used for changes within the /next/ folder.
- `docs(release): freeze version 1.2` — Used when freezing a new release version.

---

## 6. Checklist Before Merging a PR

Before submitting your PR, please complete the mandatory checklist provided in the PR template. 
Ensure all items in the PR description are checked off before requesting a review. 
