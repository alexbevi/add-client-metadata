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

### Pattern A — Library constructs the client

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

### Pattern B — Caller passes an existing client

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

# Find driver= kwarg (already set?)
pattern: driver\s*=\s*DriverInfo
glob: **/*.py

# Find append_metadata calls
pattern: append_metadata
glob: **/*.py
```

Exclude: `__pycache__/`, `.venv/`, `build/`, `dist/`, `tests/` (check tests separately).

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

### Pattern A — Library constructs the client

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

### Pattern B — Caller passes an existing client

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

### Pattern A — Library constructs the client

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

### Pattern B — Caller passes an existing client

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

### Pattern A — Library constructs the client via MongoClientSettings

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

### Pattern B — Caller passes an existing client

The C# driver does not expose a public `appendMetadata` API for existing client instances. Document this in code comments: metadata must be set at construction time via `MongoClientSettings.LibraryInfo`. Recommend callers configure this themselves, and optionally provide a helper:
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

### Pattern A — Library constructs the client

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

### Pattern B — Caller passes an existing client

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

## Cross-language: Handling mixed patterns in one file

Some files have both Pattern A and Pattern B — e.g., an optional `client` parameter where the library falls back to constructing its own:

```python
def __init__(self, client: MongoClient = None):
    if client is None:
        self._client = MongoClient(uri, driver=_DRIVER_INFO)  # Pattern A
    else:
        if hasattr(client, 'append_metadata'):                 # Pattern B
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
