# Language Patterns for MongoDB Client Metadata

Per-language grep patterns, code templates, and testing instructions.

---

## JavaScript / TypeScript

### Grep patterns

```
# Find MongoClient constructions
pattern: new MongoClient\(
glob: **/*.{ts,js,tsx,jsx}

# Find files that import MongoClient
pattern: (import|require).*MongoClient
glob: **/*.{ts,js,tsx,jsx}

# Find appendMetadata calls (to check if already done)
pattern: appendMetadata
glob: **/*.{ts,js,tsx,jsx}
```

Exclude: `node_modules/`, `dist/`, `.next/`, `coverage/`, `**/*.test.*`, `**/*.spec.*` (check tests separately).

### Version resolution

Import version from `package.json` using `resolveJsonModule`:

```typescript
import { version } from '../package.json';
// or relative path depending on file location
import { version } from '../../package.json';
```

If `tsconfig.json` does not already have `"resolveJsonModule": true`, add it to `compilerOptions`.

### Library constructs the client

**Before:**
```typescript
const client = new MongoClient(uri, options);
// or
const client = new MongoClient(uri);
```

**After:**
```typescript
import { version } from '../package.json';

// module-level constant (define once, reuse across call sites)
const DRIVER_INFO = { name: 'LibraryName', version };

const client = new MongoClient(uri, { ...options, driverInfo: DRIVER_INFO });
```

If `options` is an object variable being built up before the constructor call:
```typescript
options.driverInfo = DRIVER_INFO;
const client = new MongoClient(uri, options);
```

Guard if user may have pre-supplied it:
```typescript
if (!options.driverInfo) {
  options.driverInfo = DRIVER_INFO;
}
```

### Caller passes an existing client

Call `appendMetadata` on the received client at the earliest lifecycle point (constructor body, `init()`, `connect()`, etc.). Always guard the call with a runtime availability check — `appendMetadata` was added in Node.js `mongodb` driver v6.18.0 and callers may be on an older version:

```typescript
// in constructor or init method:
if (typeof this.client.appendMetadata === 'function') {
  this.client.appendMetadata({ name: 'LibraryName' });
}
// version is optional here since the client was constructed externally
```

`appendMetadata` is idempotent and safe to call multiple times; the driver deduplicates entries.

### tsconfig.json change (if needed)

```json
{
  "compilerOptions": {
    "resolveJsonModule": true
  }
}
```

### Testing (JS/TS)

If there are existing tests that construct or mock `MongoClient`, look for them under `__tests__/`, `test/`, or `*.spec.ts` files.

Check for:
- `MongoClient` mock setups (`jest.mock`, `vi.mock`, `sinon.stub`)
- Assertions on `MongoClientOptions` passed to the constructor
- Tests that assert on connection behavior

Propose a test that verifies `driverInfo` is set. Example (Jest):
```typescript
it('sets MongoDB driverInfo', () => {
  const clientSpy = jest.spyOn(MongoClient.prototype, 'constructor');
  // or inspect the options object passed to the constructor
  const options = getMongoOptions(); // helper that returns what would be passed
  expect(options.driverInfo).toEqual({ name: 'LibraryName', version: expect.any(String) });
});
```

**Local test run:**
```bash
cd projects/<name>
npm install        # or yarn / pnpm
npm test           # or the test script from package.json
```

---

## Python

### Grep patterns

```
# Find MongoClient / AsyncMongoClient constructions
pattern: (MongoClient|AsyncMongoClient)\s*\(
glob: **/*.py

# Find Motor AsyncIOMotorClient (excluded from injection — see note below)
pattern: AsyncIOMotorClient\s*\(
glob: **/*.py

# Find driver= kwarg (already set?)
pattern: driver\s*=\s*DriverInfo
glob: **/*.py

# Find append_metadata calls
pattern: append_metadata
glob: **/*.py
```

Exclude: `__pycache__/`, `.venv/`, `build/`, `dist/`, `tests/` (check tests separately).

> **Motor exclusion**: `AsyncIOMotorClient` (from the `motor` package) does not accept a `driver=` parameter. Only inject on PyMongo's `MongoClient` and `AsyncMongoClient`. When a codebase uses both, check the import — `from motor.motor_asyncio import AsyncIOMotorClient` is Motor; `from pymongo import AsyncMongoClient` is PyMongo async.

### Version resolution

```python
from importlib.metadata import version as get_version
# use the PyPI package name (as listed on PyPI, not import name)
_VERSION = get_version("package-name")
```

This works at runtime without hardcoding. Wrap in a try/except if the package might not be installed in all environments:
```python
try:
    from importlib.metadata import version as get_version
    _VERSION = get_version("package-name")
except Exception:
    _VERSION = None
```

### Library constructs the client

**Before:**
```python
client = MongoClient(uri)
# or
client = AsyncMongoClient(uri, **kwargs)
```

**After:**
```python
from importlib.metadata import version as get_version
from pymongo.driver_info import DriverInfo

_DRIVER_INFO = DriverInfo(name="LibraryName", version=get_version("package-name"))

client = MongoClient(uri, driver=_DRIVER_INFO)
# or
client = AsyncMongoClient(uri, driver=_DRIVER_INFO, **kwargs)
```

Define `_DRIVER_INFO` at module level, not inside the function, to avoid re-instantiating it on every call.

### Caller passes an existing client

Call `append_metadata` on the received client at the first point it is used. Always guard the call with `hasattr` — `append_metadata` was added in PyMongo 4.14 and callers may be on an older version:

```python
from pymongo.driver_info import DriverInfo
from importlib.metadata import version as get_version

_DRIVER_INFO = DriverInfo(name="LibraryName", version=get_version("package-name"))

# in __init__ or an init/connect method:
if hasattr(client, 'append_metadata'):
    client.append_metadata(_DRIVER_INFO)
```

`DriverInfo` is available from PyMongo ≥ 4.0. `append_metadata` was added in PyMongo ≥ 4.14.

### Testing (Python)

Look for test files matching `test_*.py` or `*_test.py`. Search for tests that construct `MongoClient` or mock it.

Propose a test that verifies the `driver` kwarg is passed. Example (pytest):
```python
from unittest.mock import patch, MagicMock
from pymongo.driver_info import DriverInfo

def test_mongo_client_driver_info():
    with patch("mylib.module.MongoClient") as mock_client:
        MyLib.init("mongodb://localhost")
        call_kwargs = mock_client.call_args.kwargs
        assert "driver" in call_kwargs
        assert isinstance(call_kwargs["driver"], DriverInfo)
        assert call_kwargs["driver"].name == "LibraryName"
```

**Local test run:**
```bash
cd projects/<name>
pip install -e ".[dev]"   # or pip install -e ".[test]" depending on extras
pytest
```

---

## Ruby

### Grep patterns

```
# Find Mongo::Client constructions
pattern: Mongo::Client\.new\(
glob: **/*.rb

# Find wrapping_libraries (already set?)
pattern: wrapping_libraries
glob: **/*.rb
```

Exclude: `vendor/`, `spec/`, `test/` (check separately).

### Version resolution

Look for an existing version constant in the codebase:
```
pattern: VERSION\s*=
glob: **/*.rb
```

Common locations: `lib/<gem-name>/version.rb`, then referenced as `MyLib::VERSION` or just `VERSION`.

### Library constructs the client

Ruby has no `appendMetadata` post-construction API. Metadata must be passed to the constructor as the `wrapping_libraries:` option.

**Before:**
```ruby
client = Mongo::Client.new(addresses_or_uri, options)
```

**After:**
```ruby
MONGOMAPPER_WRAPPING_LIBRARY = {
  name: 'LibraryName',
  version: MyLib::VERSION,
}.freeze

options = options.merge(wrapping_libraries: [MONGOMAPPER_WRAPPING_LIBRARY])
client = Mongo::Client.new(addresses_or_uri, options)
```

Or if `options` is a plain hash and you can modify it before passing:
```ruby
options[:wrapping_libraries] = [{ name: 'LibraryName', version: MyLib::VERSION }]
client = Mongo::Client.new(addresses_or_uri, options)
```

Define the constant at class/module level, not inside the method.

### Caller passes an existing client

The Ruby driver does not expose a public `append_metadata`-style API for existing client instances. For this pattern, document in code comments that the library cannot add metadata to externally-created clients, and recommend users pass the `wrapping_libraries` option when constructing their client.

### Testing (Ruby)

Look for `spec/` or `test/` directories. Search for tests that create `Mongo::Client`:
```
pattern: Mongo::Client\.new
glob: spec/**/*.rb
```

Propose a test (RSpec):
```ruby
describe 'MongoDB client metadata' do
  it 'includes wrapping_libraries in client options' do
    uri = 'mongodb://localhost'
    expect(Mongo::Client).to receive(:new) do |_uri, opts|
      expect(opts[:wrapping_libraries]).to include(
        hash_including(name: 'LibraryName')
      )
      instance_double(Mongo::Client)
    end
    MyLib.connect(uri)
  end
end
```

**Local test run:**
```bash
cd projects/<name>
bundle install
bundle exec rspec       # or bundle exec rake spec
```

---

## C# (.NET)

### Grep patterns

```
# Find MongoClient constructions
pattern: new MongoClient\(
glob: **/*.cs

# Find MongoClientSettings usage
pattern: MongoClientSettings
glob: **/*.cs

# Find LibraryInfo (already set?)
pattern: LibraryInfo
glob: **/*.cs
```

Exclude: `bin/`, `obj/`, `*Tests*/` (check tests separately).

### Version resolution

The recommended approach is to read the assembly version at runtime:
```csharp
using System.Reflection;

private static readonly string _version =
    typeof(MyLibClass).Assembly.GetName().Version?.ToString() ?? "unknown";
```

Or embed it from the `.csproj`:
```xml
<PropertyGroup>
  <Version>1.2.3</Version>
</PropertyGroup>
```
```csharp
// In a version constant file:
internal static class BuildInfo
{
    public const string Version = "1.2.3"; // keep in sync with .csproj
}
```

### Library constructs the client via MongoClientSettings

**Before:**
```csharp
var settings = MongoClientSettings.FromConnectionString(connectionString);
var client = new MongoClient(settings);
// or
var client = new MongoClient(connectionString);
```

**After:**
```csharp
using MongoDB.Driver.Core.Configuration;

private static LibraryInfo GetLibraryInfo() =>
    new LibraryInfo("LibraryName", _version);

// When constructing via settings:
var settings = MongoClientSettings.FromConnectionString(connectionString);
settings.LibraryInfo = GetLibraryInfo();
var client = new MongoClient(settings);

// When constructing directly from URI string, convert first:
var settings = MongoClientSettings.FromUrl(new MongoUrl(connectionString));
settings.LibraryInfo = GetLibraryInfo();
var client = new MongoClient(settings);
```

`LibraryInfo` is available in the C# driver ≥ 2.20.0. There is no post-construction append API in C#.

### Caller passes an existing client

The C# driver does not expose a public `appendMetadata` API for existing client instances. `MongoClientSettings` is frozen by the driver on `new MongoClient(settings)`, so `client.Settings.LibraryInfo` throws after construction. Document this in code comments: metadata must be set at construction time via `MongoClientSettings.LibraryInfo`. Recommend callers configure this themselves, and optionally provide a helper:
```csharp
/// <summary>
/// Returns a <see cref="LibraryInfo"/> for use with <see cref="MongoClientSettings.LibraryInfo"/>
/// when constructing a <see cref="MongoClient"/> for use with LibraryName.
/// </summary>
public static LibraryInfo GetMongoLibraryInfo() =>
    new LibraryInfo("LibraryName", _version);
```

### Testing (C#)

Look under `*Tests*/`, `*.Tests/`, or `test/` directories. Search for tests that instantiate `MongoClient` or `MongoClientSettings`.

Propose a test (xUnit/NUnit):
```csharp
[Fact]
public void MongoClient_HasLibraryInfo()
{
    var settings = MyLib.GetMongoClientSettings("mongodb://localhost");
    Assert.NotNull(settings.LibraryInfo);
    Assert.Equal("LibraryName", settings.LibraryInfo.Name);
}
```

**Local test run:**
```bash
cd projects/<name>
dotnet build
dotnet test
```

---

## Java

### Grep patterns

```
# Find MongoClients.create calls
pattern: MongoClients\.create\(
glob: **/*.java

# Find MongoClient constructor calls (legacy driver)
pattern: new MongoClient\(
glob: **/*.java

# Find MongoDriverInformation (already set?)
pattern: MongoDriverInformation
glob: **/*.java
```

Exclude: `target/`, `build/`, `*Test*.java`, `*Spec*.java` (check tests separately).

### Version resolution

Prefer a generated `BuildConfig` class if the build system produces one (common in Gradle/Maven plugins that emit source):

```java
// Generated by build — contains NAME and VERSION constants
import com.example.BuildConfig;

MongoDriverInformation.builder()
    .driverName(BuildConfig.NAME)
    .driverVersion(BuildConfig.VERSION)
    .build();
```

If no `BuildConfig` exists, read from the package manifest or a version constant:

```java
String version = MyLibClass.class.getPackage().getImplementationVersion();
// version may be null outside a packaged JAR; omit rather than pass null
```

Or reference an existing version constant already in the codebase (search for `VERSION` or `getVersionString()`).

### Library constructs the client

Define a static constant and pass it as the second argument to `MongoClients.create()`. This works for both sync and reactive clients:

```java
import com.mongodb.MongoDriverInformation;

// module-level constant
private static final MongoDriverInformation DRIVER_INFO = MongoDriverInformation.builder()
        .driverName("LibraryName")
        .driverVersion(VERSION) // omit if not reliably available
        .build();

// Sync client (com.mongodb.client)
com.mongodb.client.MongoClients.create(settings, DRIVER_INFO);

// Reactive client (com.mongodb.reactivestreams.client)
com.mongodb.reactivestreams.client.MongoClients.create(settings, DRIVER_INFO);
```

For legacy code using the `MongoClient` constructor directly, pass `MongoDriverInformation` as the last argument:

```java
new MongoClient(serverAddresses, credential, clientOptions,
    MongoDriverInformation.builder()
        .driverName("LibraryName")
        .driverVersion(VERSION)
        .build());
```

### Caller passes an existing client

`MongoClient.appendMetadata()` was added in Java driver 5.6.0. Always use reflection to guard the call so the library works with older driver versions:

```java
import com.mongodb.MongoDriverInformation;
import java.lang.reflect.Method;

private static final MongoDriverInformation DRIVER_INFO = MongoDriverInformation.builder()
        .driverName("LibraryName")
        .build();

private static void appendMongoClientMetadata(MongoClient mongoClient) {
    try {
        Method method = mongoClient.getClass().getMethod("appendMetadata", MongoDriverInformation.class);
        method.invoke(mongoClient, DRIVER_INFO);
    } catch (Exception e) {
        // appendMetadata not available in this driver version — skip silently
    }
}
```

Call `appendMongoClientMetadata(client)` at the earliest lifecycle point (constructor body, `init()`, `configure()`, etc.).

### Testing (Java)

Look under `src/test/`, `*Test*.java`, or `*IT*.java` files. Search for tests that construct `MongoClient` or `MongoClientSettings`.

Propose a test (JUnit 5):
```java
@Test
void mongoClient_hasDriverInfo() {
    MongoClientSettings settings = MyLib.buildMongoClientSettings("mongodb://localhost");
    // Inspect settings or capture the MongoDriverInformation passed to MongoClients.create
    assertNotNull(settings); // extend based on how the library exposes settings
}
```

**Local test run:**
```bash
cd projects/<name>
./mvnw test        # Maven
# or
./gradlew test     # Gradle
```

---

## Rust

### Grep patterns

```
# Find Client::with_options calls (client construction)
pattern: Client::with_options\(
glob: **/*.rs

# Find ClientOptions::parse calls
pattern: ClientOptions::parse
glob: **/*.rs

# Find driver_info (already set?)
pattern: driver_info
glob: **/*.rs
```

Exclude: `target/`, `**/tests/`, `*_test.rs` (check tests separately).

### Version resolution

Use the `env!` macro — resolved at compile time from `Cargo.toml`, always accurate:

```rust
const VERSION: &str = env!("CARGO_PKG_VERSION");
```

### Library constructs the client

The library calls `ClientOptions::parse` and `Client::with_options` itself.

**Before:**
```rust
let mut options = ClientOptions::parse(uri).await?;
let client = Client::with_options(options)?;
```

**After:**
```rust
use mongodb::options::{ClientOptions, DriverInfo};

const DRIVER_NAME: &str = "LibraryName";

pub async fn create_client(uri: &str) -> Result<Client, mongodb::error::Error> {
    let mut options = ClientOptions::parse(uri).await?;
    options.driver_info = Some(
        DriverInfo::builder()
            .name(DRIVER_NAME)
            .version(env!("CARGO_PKG_VERSION"))
            .build(),
    );
    Client::with_options(options)
}
```

Define `DRIVER_NAME` as a module-level constant. `version` is optional but recommended when set via `env!`.

### Caller passes an existing client

The Rust mongodb driver has no post-construction metadata API — `driver_info` must be set on `ClientOptions` before `Client::with_options` is called. You cannot mutate a `Client` after construction.

The correct approach is to expose a public helper so callers can build a properly-configured client:

```rust
use mongodb::options::{ClientOptions, DriverInfo};

const DRIVER_NAME: &str = "LibraryName";

/// Returns `ClientOptions` pre-configured with LibraryName driver metadata.
/// Use this when constructing a [`mongodb::Client`] for use with this library.
pub async fn client_options(uri: &str) -> Result<ClientOptions, mongodb::error::Error> {
    let mut options = ClientOptions::parse(uri).await?;
    options.driver_info = Some(
        DriverInfo::builder()
            .name(DRIVER_NAME)
            .build(),
    );
    Ok(options)
}
```

Callers then do:
```rust
let options = my_lib::client_options("mongodb://localhost").await?;
let client = Client::with_options(options)?;
```

If the library already has a builder or init function, inject the `driver_info` assignment there rather than adding a standalone function.

### Testing (Rust)

Look under `tests/`, `src/` (inline `#[cfg(test)]` modules), or integration test files. Search for tests that call `ClientOptions::parse` or `Client::with_options`.

Propose a test that verifies `driver_info` is set:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn client_options_sets_driver_info() {
        let options = client_options("mongodb://localhost").await.unwrap();
        let info = options.driver_info.expect("driver_info should be set");
        assert_eq!(info.name, "LibraryName");
    }
}
```

**Local test run:**
```bash
cd projects/<name>
cargo test
```

---

## C++

Requires mongo-cxx-driver ≥ 4.5.0 (CXX-3274). Unlike the other languages here, mongocxx has no construction-time metadata field — there is only one API, `append_metadata`, and both integration approaches call it; they differ only in *where* it's called.

### Grep patterns

```
# Find mongocxx::client / mongocxx::pool constructions
pattern: mongocxx::(client|pool)\s*\{|\bnew\s+mongocxx::(client|pool)\(
glob: **/*.{cpp,cc,cxx,hpp,h}

# Find append_metadata calls (already done?)
pattern: append_metadata
glob: **/*.{cpp,cc,cxx,hpp,h}
```

Exclude: `build/`, `cmake-build-*/`, `**/test/`, `**/tests/` (check tests separately).

### Version resolution

Prefer a version already exposed by the project's own build (a CMake-configured header, `PROJECT_VERSION`, or an existing version constant). If none exists, a hardcoded string with a comment noting where to keep it in sync (e.g. next to the `project(... VERSION x.y.z)` line in `CMakeLists.txt`) is acceptable — C++ builds are typically version-pinned rather than resolved at runtime.

### Library constructs the client

`append_metadata` is a post-construction call, so append it immediately after construction — a lambda initializer keeps this in the member-initializer list without a second statement:

**Before:**
```cpp
mongocxx::client client{uri};
```

**After:**
```cpp
mongocxx::client client{[&uri]
    {
        mongocxx::client c{uri};
        c.append_metadata("LibraryName", LIBRARY_VERSION);
        return c;
    }()};
```

Give the lambda-local variable a name distinct from any enclosing member named `client` — clang-tidy's `-Wshadow-uncaptured-local` (common in `-Werror` builds) flags a lambda-local that shadows a class member of the same name.

If the constructor body runs before any operation touches the server, a plain statement after construction works too and is simpler when there's no member-initializer-list constraint:
```cpp
mongocxx::client client{uri};
client.append_metadata("LibraryName", LIBRARY_VERSION);
```

The same API and pattern apply to `mongocxx::pool`.

`append_metadata` throws `mongocxx::exception` (or `mongocxx::operation_exception` under the stable ABI) if the resulting handshake document would exceed the size limit, or if a `name`/`version`/`platform` argument contains the literal substring `" / "` (the driver's own delimiter). Let it propagate unless the surrounding code already has a broader exception-handling convention to fold it into.

### Caller passes an existing client

Call `append_metadata` on the received `mongocxx::client&` or `mongocxx::pool&` reference at the earliest point the library takes ownership or first uses it:

```cpp
void MyLib::attach(mongocxx::client& client) {
    client.append_metadata("LibraryName", LIBRARY_VERSION);
    _client = &client;
}
```

There is no availability guard needed comparable to other languages' `hasattr`/reflection checks: `append_metadata` is a compile-time API, so if the project builds against mongocxx ≥ 4.5.0 the symbol is always present. If the project must support both older and newer driver versions, guard with the driver's version macros instead:
```cpp
#include <mongocxx/config/version.hpp>

#if MONGOCXX_VERSION_MAJOR > 4 || (MONGOCXX_VERSION_MAJOR == 4 && MONGOCXX_VERSION_MINOR >= 5)
    client.append_metadata("LibraryName", LIBRARY_VERSION);
#endif
```

### Testing (C++)

Look under `test/`, `tests/`, or files matching `*_test.cpp`/`*Test.cpp`. Search for tests that construct `mongocxx::client` or `mongocxx::pool`.

Propose a test that verifies metadata is appended without throwing (most test suites don't have a way to inspect the outgoing handshake document directly, so the test typically just confirms the call site is exercised and doesn't throw):

```cpp
TEST_CASE("MongoDB client sets library metadata") {
    mongocxx::instance instance{};
    mongocxx::uri uri{"mongodb://localhost/?connectTimeoutMS=1"};
    REQUIRE_NOTHROW(my_lib::create_client(uri));
}
```

**Local test run:**
```bash
cd projects/<name>
cmake --preset default   # or the project's documented configure step
cmake --build build --target test
```

---

## C

Requires mongo-c-driver (libmongoc) ≥ 2.3.0. As with C++, there is a single post-construction API — `mongoc_client_append_metadata` / `mongoc_client_pool_append_metadata` — used by both integration approaches; there is no construction-time metadata field.

### Grep patterns

```
# Find mongoc_client_new / mongoc_client_pool_new constructions
pattern: mongoc_client(_pool)?_new\s*\(
glob: **/*.{c,h}

# Find append_metadata calls (already done?)
pattern: mongoc_client(_pool)?_append_metadata
glob: **/*.{c,h}
```

Exclude: `build/`, `cmake-build-*/`, `test/`, `tests/` (check tests separately).

### Version resolution

Same as C++: prefer a version already generated by the build (an autoconf/CMake-configured header or existing `PACKAGE_VERSION`-style macro). Otherwise a hardcoded string kept next to the build's version declaration is acceptable.

### Library constructs the client

**Before:**
```c
mongoc_client_t *client = mongoc_client_new_from_uri(uri);
```

**After:**
```c
mongoc_client_t *client = mongoc_client_new_from_uri(uri);
if (client) {
    mongoc_client_append_metadata(client, "LibraryName", LIBRARY_VERSION, NULL);
}
```

For pooled clients:
```c
mongoc_client_pool_t *pool = mongoc_client_pool_new(uri);
mongoc_client_pool_append_metadata(pool, "LibraryName", LIBRARY_VERSION, NULL);
```

`mongoc_client_append_metadata`/`mongoc_client_pool_append_metadata` return `bool` and log an error rather than aborting on failure (e.g. the resulting handshake document would exceed the size limit, or a `name`/`version`/`platform` argument contains the literal substring `" / "`). Check the return value if the project has a convention of propagating driver errors; otherwise logging is the driver's own fallback and no additional handling is required.

### Caller passes an existing client

Call the same API on the received `mongoc_client_t *` or `mongoc_client_pool_t *` at the earliest point the library takes ownership or first uses it:

```c
void my_lib_attach(mongoc_client_t *client) {
    mongoc_client_append_metadata(client, "LibraryName", LIBRARY_VERSION, NULL);
}
```

If a `mongoc_client_t *` obtained from `mongoc_client_pool_pop` is passed in, call `mongoc_client_pool_append_metadata` on the *pool* instead — `mongoc_client_append_metadata` explicitly rejects (returns `false`, logs an error) clients checked out from a pool.

No runtime availability guard is needed if the project's build requires mongo-c-driver ≥ 2.3.0. To support older driver versions in the same codebase, guard with the driver's version macro instead:
```c
#include <mongoc/mongoc-version.h>

#if MONGOC_CHECK_VERSION(2, 3, 0)
    mongoc_client_append_metadata(client, "LibraryName", LIBRARY_VERSION, NULL);
#endif
```

### Testing (C)

Look under `tests/` or files matching `test-*.c`. Search for tests that call `mongoc_client_new*` or `mongoc_client_pool_new*`.

Propose a test that verifies the call succeeds:

```c
static void
test_my_lib_sets_metadata (void) {
   mongoc_client_t *client = mongoc_client_new ("mongodb://localhost/?connectTimeoutMS=1");
   my_lib_attach (client);
   /* No public API exposes the outgoing handshake document for inspection;
      asserting the call doesn't abort is the achievable coverage here. */
   mongoc_client_destroy (client);
}
```

**Local test run:**
```bash
cd projects/<name>
cmake --preset default   # or the project's documented configure step
cmake --build build --target test
```

---

## Cross-language: Handling both integration approaches in one file

Some files construct their own client in one branch and accept a caller-supplied client in another — e.g., an optional `client` parameter where the library falls back to constructing its own:

```python
def __init__(self, client: MongoClient = None):
    if client is None:
        self._client = MongoClient(uri, driver=_DRIVER_INFO)  # library constructs the client
    else:
        if hasattr(client, 'append_metadata'):                 # caller passed an existing client
            client.append_metadata(_DRIVER_INFO)
        self._client = client
```

Handle both branches in the same edit.

## Determining the display name

Use the following heuristics for the `name` field:
1. Check if there's a human-readable display name constant in the codebase (e.g. `DISPLAY_NAME = "Beanie ODM"`).
2. Otherwise, use the package name from the manifest, title-cased sensibly:
   - `langchain-mongodb` → `"Langchain"`
   - `flask-pymongo` → `"Flask-PyMongo"`
   - `typeorm` → `"TypeORM"` (check the project README/website for canonical casing)
   - `beanie` → `"Beanie"`
3. Keep the name short and recognizable. Avoid version numbers in the name.
