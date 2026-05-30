# Sanity Criteria — docs

> Verify XML documentation comments and markdown documentation are complete and current.
> Read `architecture/naming-conventions.md` before evaluating.

---

## XML Documentation Comments

### Language

- All XML comments are written in **English**.

### Required tags

| Tag | Rule |
|-----|------|
| `<summary>` | **Always required** on every documented member. One or two sentences max. |
| `<param>` | Omit when name + type make the purpose obvious. Write when semantics are non-trivial. |
| `<returns>` | Omit for `void`. Omit when the return value mirrors the summary exactly. Write when the caller needs to know what to expect beyond the obvious. |
| `<remarks>` | Only when genuinely helpful context would not fit in `<summary>`. Rare. |
| `<exception>` | **Never used** — failures are expressed via `Result` / `Result<T>`, never thrown. |
| `<inheritdoc/>` | Always on implementations that formally implement a documented interface or base. Never duplicate the contract text. |

### Where comments live

- **Interfaces / contracts** carry the full XML documentation.
- **Implementations** use `<inheritdoc/>` (or `/// <inheritdoc cref="IXxx.Member"/>` for static classes).
- A member must never be documented in two places.

---

## Scope by Project Type

### NuGet library projects

- All **public** members must be documented — `GenerateDocumentationFile` is `true`.
- **Internal** members: document only when context is genuinely non-obvious to a maintainer. Be thrifty.
- Verify `<GenerateDocumentationFile>true</GenerateDocumentationFile>` is present in the csproj.

### Exe and non-NuGet projects

- `<GenerateDocumentationFile>` must be **absent** from the csproj.
- Document public and internal members only when the purpose is not obvious from name + context alone.

### All project types — interface rule

- **Interfaces must always be fully XML-documented**, regardless of project type. No exceptions.

---

## Quality Checklist

For each file in scope, flag any of the following as a violation:

| Signal | Verdict |
|--------|---------|
| Missing `<summary>` on a documented member | ❌ violation |
| Empty `<summary/>` or `// TODO` placeholder | ❌ violation |
| Comment restates the method name word-for-word | ❌ violation |
| Comment references a renamed type or old namespace | ❌ violation |
| `<remarks>` with a single obvious sentence | ❌ violation |
| Implementation duplicates interface doc text instead of using `<inheritdoc/>` | ❌ violation |
| `<exception>` tag present anywhere | ❌ violation |
| NuGet project missing `<GenerateDocumentationFile>true</GenerateDocumentationFile>` | ❌ violation |
| Exe/test project has `<GenerateDocumentationFile>` present | ❌ violation |
| Good, clear, concise summary | ✅ pass |

---

## Markdown Documentation

- Every repo has a `README.md` at the root.
- README content accurately reflects the current state of the repo (assemblies, packages, entry points).
- No references to renamed types, old namespaces, or removed features.
- Cross-repo links (e.g. `../josyn-foundation/README.md`) resolve to existing files.
