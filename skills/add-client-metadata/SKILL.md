---
name: add-client-metadata
description: >
  Add MongoDB driver handshake metadata to a third-party library that uses MongoClient.
  Use when the user invokes "add-client-metadata <repo-url>" or asks to add MongoDB client
  metadata, driver info, or handshake metadata to a library or GitHub repository.
version: 0.5.0
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

### Step 0 — Pre-flight checks

Before cloning, run two fast checks to avoid wasted work.

**Check A — Already done?**

Search the repo on GitHub (or via the raw URL) for existing driver metadata. If any of the following are already present (and not commented out), the work is done — report this to the user and stop:

- Python: `append_metadata`, `DriverInfo`, `driver=DriverInfo`
- JS/TS: `appendMetadata`, `driverInfo:`
- Ruby: `wrapping_libraries:`
- C#: `LibraryInfo`
- Java/Kotlin: `MongoDriverInformation`
- Rust: `DriverInfo::builder`
- C++: `append_metadata` on a `mongocxx::client`/`mongocxx::pool`
- C: `mongoc_client_append_metadata` or `mongoc_client_pool_append_metadata`

Also check for commented-out driver code (e.g. `# driver=DriverInfo(...)`, `// driver=...`). If found, note it — the implementation may just need to be uncommented and imports added rather than written from scratch.

**Check B — MongoDB used at all?**

Before cloning, check that the repo actually uses MongoDB. Search for `mongo` (case-insensitive) in the repo's file listing or README. If there are no MongoDB references, report this to the user and stop — do not clone.

### Step 1 — Clone the repository

Derive the project directory name from the last path segment of the URL (e.g. `github.com/foo/my-lib` → `my-lib`).

```bash
mkdir -p projects
git clone <url> projects/<name>
```

If the directory already exists, pull the latest changes instead of re-cloning.

### Step 2 — Read contribution guidelines

Before writing any code, check for contribution and code-quality requirements:

- Look for `CONTRIBUTING.md`, `CONTRIBUTING.rst`, `.github/CONTRIBUTING.md`, or similar files.
- Check for a `.pre-commit-config.yaml` to identify the exact lint hooks (e.g. black, isort, flake8, eslint, prettier).
- Check `pyproject.toml`, `package.json`, or `Makefile` for lint/test scripts.
- Note any specific rules about commit message format, PR scope, or import ordering.

Record the lint command(s) and test command(s) — you will need them in Steps 6 and 8.

### Step 3 — Detect language

Check for the presence of these files in `projects/<name>/`:

| File | Language |
|---|---|
| `package.json` | JavaScript / TypeScript |
| `pyproject.toml` or `setup.py` or `requirements.txt` | Python |
| `Gemfile` or `*.gemspec` | Ruby |
| `*.csproj` or `*.sln` | C# |
| `pom.xml` or `build.gradle` or `build.gradle.kts` | Java / Kotlin |
| `Cargo.toml` | Rust |
| `CMakeLists.txt` or `conanfile.txt`/`conanfile.py` linking `mongocxx` | C++ |
| `CMakeLists.txt` or a vcpkg/Conan manifest linking `mongoc`/`libmongoc` (no `mongocxx`) | C |

A repo may mix languages. Apply changes in all relevant language contexts. If a build links both `mongoc` and `mongocxx`, treat it as C++ — `mongocxx` wraps `mongoc`, and metadata should be appended once via the C++ API rather than duplicated at both layers.

### Step 4 — Identify library name and version

**Library name**: Use the package name from the manifest:
- JS/TS: `name` field in `package.json`
- Python: `name` in `pyproject.toml` or `setup.py`
- Ruby: gem name from `*.gemspec` or `Gemfile` project name
- C#: `<AssemblyName>` or `<PackageId>` from `.csproj`
- Java/Kotlin: `artifactId` from `pom.xml` or `rootProject.name` from `settings.gradle`
- Rust: `name` field in `Cargo.toml` under `[package]`
- C++/C: `project(...)` name in `CMakeLists.txt`, or the package `name` in a Conan/vcpkg manifest

Capitalize the library name sensibly for display (e.g. `langchain-mongodb` → `"Langchain"`, `beanie` → `"Beanie"`, `typeorm` → `"TypeORM"`). Check whether the existing codebase has a display-name constant already; if so, use it.

**Library version**: See `references/language-patterns.md` for how to read version at runtime per language.

### Step 5 — Scan for MongoClient integration points

Use Grep to find all `MongoClient` construction and usage sites. Consult `references/language-patterns.md` for per-language grep patterns and file globs. For Rust, search for `ClientOptions::parse` and `Client::with_options`. For C++, search for `mongocxx::client` and `mongocxx::pool` construction. For C, search for `mongoc_client_new` and `mongoc_client_pool_new`.

For each hit, determine which integration approach applies:

**Library constructs the client** — The library calls `new MongoClient(...)` (or equivalent) directly. The URL / connection string is typically a library config value.
→ Inject `driverInfo` into the constructor call.

**Caller passes an existing client** — The library accepts a `MongoClient` instance as a parameter (function argument, constructor argument, or config field). The `new MongoClient` call lives in user code, outside this library.
→ Call the `appendMetadata` / `append_metadata` API on the received client instance, typically at the earliest point the client is used (e.g. in an `init`, `connect`, or `setup` method).

A single library may use both approaches (e.g. it accepts an optional existing client but also creates one if none is provided). Handle both branches.

**Python async note:** Motor's `AsyncIOMotorClient` does not support the `driver=` parameter — only inject on PyMongo's `MongoClient` and `AsyncMongoClient`. If a codebase uses both Motor and PyMongo async clients, inject only on the PyMongo async path.

**C++ / C note:** `mongocxx`/`mongoc` have no construction-time metadata field — both integration approaches call the same post-construction `append_metadata` API (on `mongocxx::client`/`mongocxx::pool`, or `mongoc_client_append_metadata`/`mongoc_client_pool_append_metadata`). The only difference between the two approaches is *where* the call happens: immediately after the library's own constructor call, versus in the caller-supplied-client's init path.

Throughout this skill, refer to these two cases by their plain descriptions ("library constructs the client" / "caller passes an existing client") — do not invent internal shorthand like letter- or number-coded pattern names. Code comments, commit messages, and PR descriptions must describe behavior in terms the library maintainer will recognize, not skill-internal jargon.

### Step 6 — Apply code changes

Read each file that needs changing. Apply edits using idiomatic patterns for the language. Consult `references/language-patterns.md` for exact code templates.

Key principles:
- Prefer a `driverInfo` / `DriverInfo` constant defined once at module level and reused across all call sites.
- Don't overwrite user-supplied `driverInfo` if the caller already set one (use a guard like `if (!options.driverInfo)`).
- Include the library version if it is easy to read at runtime (see reference). Omit it rather than hardcode a string.
- If a tsconfig needs `resolveJsonModule: true` to import `package.json`, add it.

### Step 7 — Run lint and fix any failures

Run the lint tool(s) identified in Step 2 against the changed files. Fix every reported violation before proceeding — do not skip or suppress.

Common failure patterns and fixes:
- **Import order (isort, eslint import/order)**: reorder imports alphabetically or per the tool's grouping rules.
- **Formatting (black, prettier)**: apply the formatter to auto-fix, then verify with `--check`.
- **Unused imports (flake8 F401, eslint no-unused-vars)**: remove any import that is no longer referenced.
- **Line length**: break long lines to stay within the project's configured limit.

Re-run lint until it reports no errors. If a fix causes a new failure, resolve that too.

### Step 8 — Write tests

Add tests that verify the metadata is applied. Match the repo's existing test conventions (framework, file location, mock style).

Tests must cover at minimum:
- The `driverInfo` / `DriverInfo` constant has the expected `name` field.
- The client is constructed (or `appendMetadata` is called) with that constant.
- A caller-supplied driver value is not overridden (guard for the library-constructs-client case).
- `append_metadata` / `appendMetadata` absence does not raise (guard for the caller-supplied-client case, if applicable).

Run the tests and fix any failures before committing. See `references/language-patterns.md` for per-language testing guidance.

### Step 9 — Run lint again on test files

Run the same lint tools from Step 7 against the new test files. Fix any violations. Do not commit with lint errors.

### Step 10 — Show the diff

```bash
cd projects/<name> && git diff
```

Display the full diff to the user.

### Step 11 — Ask for confirmation

Ask the user: **"Shall I commit these changes to a new branch?"**

Use `AskUserQuestion` with options **Yes** and **No**.

### Step 12 — Commit (if confirmed)

First, check whether the project requires a DCO sign-off. Look for `Signed-off-by` mentions in `CONTRIBUTING.md`, a `.github/dco.yml` file, or a DCO check in `.github/workflows/`. If sign-off is required, add `-s` to the commit command.

```bash
cd projects/<name>
git checkout -b add-client-metadata
git add <changed files>
git commit -s -m "feat: add MongoDB driver handshake metadata"
# Omit -s if the project does not require DCO sign-off
```

Run lint one final time after staging to confirm the committed state is clean.

If the user answers No, leave the working tree with the proposed edits applied but uncommitted so they can review further.

### Step 13 — Draft PR title and body

Produce a PR title and body the user can copy. The body must follow the repo's PR template if one exists (check `.github/PULL_REQUEST_TEMPLATE.md`). It must include:

**Title:**
One concise imperative line, e.g.:
```
feat: add MongoDB driver handshake metadata
```

**Body sections:**

1. **Summary** — One paragraph explaining what the change does and why it matters. Frame it from the perspective of value to the maintainer and their users: server-side visibility, easier debugging, MongoDB Atlas integration.

2. **What changes** — A brief description of each modified file: whether the library constructs its own client (and `driverInfo` was injected into the constructor) or receives one from the caller (and `appendMetadata` was called), including the guard strategy. Describe the behavior in plain terms, not internal skill jargon.

3. **How it appears in the logs** — Include both the raw metadata dict and a realistic `mongod` log line. Generate the metadata by constructing a no-connect client:

   **Python:**
   ```python
   from pymongo import MongoClient
   from pymongo.driver_info import DriverInfo
   c = MongoClient('mongodb://localhost', driver=DriverInfo(name="LibraryName", version="x.y.z"), connect=False)
   import pprint; pprint.pprint(c.options.pool_options.metadata)
   ```

   Then wrap the output in a realistic `mongod` structured log envelope for the PR body:
   ```json
   {"t":{"$date":"2026-01-01T00:00:00.000+00:00"},"s":"I","c":"NETWORK","id":51800,"ctx":"conn1","msg":"client metadata","attr":{"remote":"<ip>:<port>","client":"conn1","negotiatedCompressors":[],"doc":<metadata>}}
   ```
   Replace `<metadata>` with the actual dict from the Python snippet above (with `doc` as a JSON object, not a string).

   **JS/TS:** describe the `nodejs|LibraryName` pattern and note the `driver.name` and `driver.version` fields.

   **Rust:** describe the `driver_info` field on `ClientOptions` set via `DriverInfo::builder().name("LibraryName").build()`, and note that the handshake document will include the library name under the `driver` field alongside the `mongodb` Rust driver info.

   **C++ / C:** describe the `append_metadata("LibraryName", version)` call on the `mongocxx::client`/`mongocxx::pool` (or `mongoc_client_append_metadata`/`mongoc_client_pool_append_metadata`), and note that the handshake `driver.name`/`driver.version` fields become slash-delimited (e.g. `"mongoc / mongocxx / LibraryName"`), matching the wrapping-library convention other drivers use.

4. **Spec reference** — Link to the MongoDB handshake specification:
   ```
   https://github.com/mongodb/specifications/blob/master/source/mongodb-handshake/handshake.md
   ```

5. **Testing** — What tests were added and what they verify.

6. **Checklist** — Fill in any checklist from the repo's PR template. Confirm lint passes, tests pass, changes are scoped to a single goal.

## Reference

Detailed code templates, grep patterns, and testing instructions for each language are in `references/language-patterns.md`.
