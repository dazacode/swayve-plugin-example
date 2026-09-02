# Example Plugin

The reference Swayve plugin. Its job is to be **read**, not to be useful.

It contributes a handful of invented tracks, albums and artists to Swayve's
search and browsing surfaces, and that is all it does. No network, no storage,
no credentials, no web view, no playback — which is exactly what makes it a
good first read. Everything that is left in the file is the shape of a plugin.

```
example/
├── plugin.json                 the manifest: what it can do, what it needs
├── pubspec.yaml                one dependency: the SDK
├── lib/example.dart            the entrypoint and the lifecycle
├── lib/src/catalogue.dart      the data, and normalization at the boundary
├── lib/src/providers.dart      search and catalog, with cancellation + paging
├── test/                       what all of the above is supposed to do
└── licenses/                   Apache-2.0 and the third-party notice
```

Read it in that order. It is about 300 lines of code and roughly as many lines
of commentary; you can finish it in one sitting.

The design rationale — why `permissions` is empty, why it claims no playback,
the eight things it is trying to teach, and the manifest field reference — is
in the [repository README](../README.md). This file is just the map.

## Running it

```bash
cd example
dart pub get
dart analyze          # zero issues
dart test             # 29 tests, no sockets, no Flutter
```

Validating it needs a [`swayve-plugins`](https://github.com/dazacode/swayve-plugins)
checkout, which is where the validator lives:

```bash
cd ../../swayve-plugins
dart pub get
dart run tools/validate_plugin.dart ../swayve-plugin-example/example --strict
```

There is no way to run it inside Swayve yet — the client has no plugin loader.
The SDK's fakes are what let this plugin be developed and proven in the
meantime, and that is deliberate: a plugin you can only test by installing the
app is a plugin nobody will test.

## Copying it

Copy this directory, then change three things in `plugin.json` before anything
else:

```jsonc
{
  "id": "dev.yourname.swayve.my_plugin",  // reverse-DNS, at least 3 segments
  "entrypoint": "my_plugin",              // MUST equal the directory name
  "name": "My Plugin"
}
```

Then rename the entrypoint function in `lib/`, update `examplePluginId`, and
run the validator. Forgetting to rename the directory *or* the `entrypoint` is
the most common first failure, and the validator names it precisely.

Do not carry the fixture catalogue forward. Replace `lib/src/catalogue.dart`
with a client for whatever you are actually integrating, keep the shape — raw
in, normalized out, at the boundary — and the providers above it should barely
need to change.
