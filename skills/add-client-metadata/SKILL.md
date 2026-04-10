---
name: add-client-metadata
description: >
  Add MongoDB driver handshake metadata to a third-party library that uses MongoClient.
  Use when the user invokes "add-client-metadata <repo-url>" or asks to add MongoDB client
  metadata, driver info, or handshake metadata to a library or GitHub repository.
version: 0.1.0
---

# Add MongoDB Client Metadata

Add MongoDB driver handshake metadata (`driverInfo` / `DriverInfo`) to a library that uses `MongoClient`. This surfaces the library's name and version in the MongoDB handshake, helping server-side telemetry distinguish traffic by library.

## Invocation

The user calls this skill with a GitHub repository URL as the argument:

```
add-client-metadata <github-repo-url>
```

Extract the URL from `$ARGUMENTS`.

## Workflow

Execute the following steps in order.

### Step 1 — Clone the repository

Derive the project directory name from the last path segment of the URL (e.g. `github.com/foo/my-lib` → `my-lib`).

```bash
mkdir -p projects
git clone <url> projects/<name>
```

If the directory already exists, pull the latest changes instead of re-cloning.

### Step 2 — Detect language

Check for the presence of these files in `projects/<name>/`:

| File | Language |
|---|---|
| `package.json` | JavaScript / TypeScript |
| `pyproject.toml` or `setup.py` or `requirements.txt` | Python |
| `Gemfile` or `*.gemspec` | Ruby |
| `*.csproj` or `*.sln` | C# |
| `pom.xml` or `build.gradle` or `build.gradle.kts` | Java / Kotlin |

A repo may mix languages. Apply changes in all relevant language contexts.

### Step 3 — Identify library name and version

**Library name**: Use the package name from the manifest:
- JS/TS: `name` field in `package.json`
- Python: `name` in `pyproject.toml` or `setup.py`
- Ruby: gem name from `*.gemspec` or `Gemfile` project name
- C#: `<AssemblyName>` or `<PackageId>` from `.csproj`
- Java/Kotlin: `artifactId` from `pom.xml` or `rootProject.name` from `settings.gradle`

Capitalize the library name sensibly for display (e.g. `langchain-mongodb` → `"Langchain"`, `beanie` → `"Beanie"`, `typeorm` → `"TypeORM"`). Check whether the existing codebase has a display-name constant already; if so, use it.

**Library version**: See `references/language-patterns.md` for how to read version at runtime per language.

### Step 4 — Scan for MongoClient integration points

Use Grep to find all `MongoClient` construction and usage sites. Consult `references/language-patterns.md` for per-language grep patterns and file globs.

For each hit, determine which integration pattern applies:

**Pattern A — Library constructs the client**
The library calls `new MongoClient(...)` (or equivalent) directly. The URL / connection string is typically a library config value.
→ Inject `driverInfo` into the constructor call.

**Pattern B — Caller passes an existing client**
The library accepts a `MongoClient` instance as a parameter (function argument, constructor argument, or config field). The `new MongoClient` call lives in user code, outside this library.
→ Call the `appendMetadata` / `append_metadata` API on the received client instance, typically at the earliest point the client is used (e.g. in an `init`, `connect`, or `setup` method).

A single library may have both patterns (e.g. it accepts an optional existing client but also creates one if none is provided). Handle both branches.

### Step 5 — Propose and apply code changes

Read each file that needs changing. Apply edits using idiomatic patterns for the language. Consult `references/language-patterns.md` for exact code templates.

Key principles:
- Prefer a `driverInfo` / `DriverInfo` constant defined once at module level and reused across all call sites.
- Don't overwrite user-supplied `driverInfo` if the caller already set one (use a guard like `if (!options.driverInfo)`).
- Include the library version if it is easy to read at runtime (see reference). Omit it rather than hardcode a string.
- If a tsconfig needs `resolveJsonModule: true` to import `package.json`, add it.

### Step 6 — Show the diff

```bash
cd projects/<name> && git diff
```

Display the full diff to the user.

### Step 7 — Ask for confirmation

Ask the user: **"Shall I commit these changes to a new branch?"**

Use `AskUserQuestion` with options **Yes** and **No**.

### Step 8 — Commit (if confirmed)

```bash
cd projects/<name>
git checkout -b add-client-metadata
git add <changed files>
git commit -m "feat: add MongoDB client metadata"
```

Then give the user instructions for testing locally. See `references/language-patterns.md` for per-language testing guidance.

If the user answers No, leave the working tree with the proposed edits applied but uncommitted so they can review further.

## Reference

Detailed code templates, grep patterns, and testing instructions for each language are in `references/language-patterns.md`.
