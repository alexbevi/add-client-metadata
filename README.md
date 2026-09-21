# add-client-metadata

A Claude Code plugin that adds MongoDB driver handshake metadata to third-party libraries that use `MongoClient`.

When a library wraps PyMongo, the Node.js MongoDB driver, the Ruby Mongo driver, the C# driver, the Rust mongodb crate, or the C/C++ drivers, MongoDB's server-side telemetry sees all traffic as generic driver connections. Adding `driverInfo` / `DriverInfo` / `wrapping_libraries` / `append_metadata` to the `MongoClient` construction surfaces the library's name and version in the handshake, enabling accurate server-side attribution.

## Usage

Install the plugin, then invoke the skill with a GitHub repository URL:

```
add-client-metadata <github-repo-url>
```

Claude will clone the repo, detect the language, find all `MongoClient` construction and usage sites, apply the appropriate metadata injection, show you a diff, and offer to commit the changes to a new branch.

## What it does

1. Clones the target repository
2. Detects language (Python, JS/TS, Ruby, C#, Java/Kotlin, Rust, C++, C) from manifest files
3. Identifies the library name and version
4. Scans for `MongoClient` integration points:
   - **Library constructs the client** — injects `driver=DriverInfo(...)` / `driverInfo: { name, version }` / `wrapping_libraries:` into the constructor call
   - **Caller passes an existing client** — calls `append_metadata` / `appendMetadata` on the received instance
5. Applies edits, shows a diff, and commits to an `add-client-metadata` branch on request

## Language support

| Language | API | Min driver version |
|---|---|---|
| Python | `DriverInfo` + `driver=` kwarg, or `append_metadata()` | PyMongo >= 4.0 / 4.14 |
| JavaScript / TypeScript | `driverInfo` option, or `appendMetadata()` | `mongodb` >= 6.x |
| Ruby | `wrapping_libraries:` option | `mongo` >= 2.x |
| C# | `MongoClientSettings.LibraryInfo` | driver >= 2.20.0 |
| Java / Kotlin | `MongoDriverInformation` + `MongoClients.create()`, or `appendMetadata()` | Java driver >= 4.x / 5.6.0 |
| Rust | `DriverInfo` on `ClientOptions`, exposed via a `client_options()` helper | `mongodb` crate >= 2.x |
| C++ | `append_metadata()` on `mongocxx::client` / `mongocxx::pool` | mongo-cxx-driver >= 4.5.0 |
| C | `mongoc_client_append_metadata()` / `mongoc_client_pool_append_metadata()` | mongo-c-driver (libmongoc) >= 2.3.0 |

## Installation

Clone this repo into your Claude Code plugins directory, or reference it from your Claude Code settings.
