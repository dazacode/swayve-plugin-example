# Swayve Plugin Example

The reference Swayve plugin, and the manifest reference that goes with it.

Two things live here:

| | What it is | Read it when |
|---|---|---|
| [`example/`](example) | A complete, working, tested plugin. ~300 lines of code and about as many of commentary. No network, no storage, no credentials, no playback. | You want to see how a plugin is actually built. |
| [`reference/`](reference) | Every field the plugin manifest schema defines, in one valid document. | You are writing a `plugin.json` and need to know what is available. |

The example is deliberately boring. It contributes a handful of invented
tracks, albums and artists to search and browsing, and does nothing else —
which is what makes it readable. Everything left in the file is the shape of a
plugin.

The platform itself — the SDK, the schema, the validator and the packaging
tools — lives in [`dazacode/swayve-plugins`](https://github.com/dazacode/swayve-plugins).
This repository depends on it and does not duplicate it.

## Quick start

```bash
git clone https://github.com/dazacode/swayve-plugin-example.git
cd swayve-plugin-example/example
dart pub get
dart analyze          # no issues
dart test             # 29 tests, no sockets, no Flutter
```

To start your own plugin, copy the directory and change three things in
`plugin.json` before anything else:

```jsonc
{
  "id": "dev.yourname.swayve.my_plugin",  // reverse-DNS, at least 3 segments
  "entrypoint": "my_plugin",              // MUST equal the directory name
  "name": "My Plugin"
}
```

Then rename the entrypoint function in `lib/`, update `examplePluginId`, and
run the validator from a `swayve-plugins` checkout:

```bash
dart run tools/validate_plugin.dart ../swayve-plugin-example/my_plugin --strict
```

Forgetting to rename the directory *or* the `entrypoint` is the most common
first failure. They must be identical, and `entrypoint` allows only
`[a-z][a-z0-9_]*` — no hyphens. That is why the plugin sits in `example/`
rather than at the root of a repository whose name has one.

Do not carry the fixture catalogue forward. Replace `lib/src/catalogue.dart`
with a client for whatever you are integrating, keep the shape — raw in,
normalized out, at the boundary — and the providers above it should barely
need to change.

## The manifest: what is required and what is not

Every plugin ships a `plugin.json`. Fourteen fields are required; ten are
optional. The example declares the required ones plus three cheap optionals,
which is why it is a good starting point — you add what you actually need.

### Required — all fourteen

| Field | Type | Notes |
|---|---|---|
| `schemaVersion` | integer | The manifest schema this document is written in. **Currently 7.** 1–6 still validate, but write new plugins at 7. |
| `id` | string | Reverse-DNS, three or more lowercase segments: `dev.yourname.swayve.my_plugin`. |
| `name` | string | Human-readable. No emoji. |
| `description` | string | One paragraph on what the plugin adds to Swayve. |
| `version` | string | Strict SemVer 2.0.0. |
| `author` | object | `name` required; `url` and `email` optional. |
| `license` | string | SPDX identifier, e.g. `Apache-2.0`. |
| `swayvePluginApi` | integer | SDK major API level. v1 plugins declare `1`. |
| `minimumSwayveVersion` | string | Lowest client version this runs on. |
| `runtime` | enum | `compiled` or `bundled`. |
| `platforms` | array | Any of `android`, `ios`, `windows`, `macos`, `linux`. |
| `capabilities` | array | Which provider interfaces you implement. Must be non-empty. |
| `permissions` | array | Host facilities you may touch. **May be empty — and should be, if you touch none.** |
| `entrypoint` | string | The Dart library and registration symbol. Must equal the directory name. |

### Optional — all ten

| Field | Use it when |
|---|---|
| `homepage` | You have a public page describing the plugin. |
| `repository` | Your source lives somewhere public. |
| `icon` | You ship a mark under `assets/`, `.png` or `.svg`. |
| `source` | Your plugin is a *place music comes from* and should appear as one. Declares what it can be asked for, independently of `capabilities`. |
| `media` | You want to state playback rights explicitly. Three independent booleans — never derive one from another. |
| `settings` | The host should render settings on your behalf. A plugin never draws its own settings page. |
| `network` | You make outbound requests. Lists the hostnames you may reach. |
| `session_capture` | You need a host-mediated sign-in capture. You name *what* to capture; the host owns *how*. |
| `timeouts` | Your service needs a different budget than the default. The host still applies its own hard limits. |
| `keywords` | Discovery. Lowercase. |

### The closed vocabularies

These are enums. A value outside them fails validation.

- **`capabilities`** — `search`, `catalog`, `streaming`, `metadata`, `lyrics`,
  `scrobbling`, `authentication`, `webview`, `artwork`, `playlist_read`,
  `artist_activity`, `personal_library`, `session_capture`,
  `personal_library_push`, `metadata_search`, `radio`, `visuals`
- **`permissions`** — `network`, `webview`, `external_auth`,
  `local_plugin_storage`, `clipboard`
- **`platforms`** — `android`, `ios`, `windows`, `macos`, `linux`
- **`runtime`** — `compiled`, `bundled`
- **`settings[].type`** — `string`, `bool`, `int`, `select`, `secret`
- **`source.contentTypes`** — `songs`, `albums`, `artists`, `videos`
- **`source.availability`** — `ready`, `offline`, `rate_limited`, `signed_out`, `off`

[`reference/plugin.reference.json`](reference/plugin.reference.json) uses every
one of these fields in a single document that validates against the real
schema. It is not a working plugin — it is what the schema looks like when
nothing is left out.

## What the example declares, and why

| Field | Value | Why |
|---|---|---|
| `capabilities` | `search`, `catalog` | Two capabilities, so exactly two providers get registered. |
| `permissions` | *(empty)* | It reads a constant compiled into itself. No host, no disk, no socket. |
| `runtime` | `compiled` | Built into Swayve rather than downloaded. |
| `platforms` | all five | Nothing here is platform-specific. |
| `media.streamable` | `false` | It resolves no playback. |
| `media.downloadable` | `false` | It has no files to hand over. |

### Why `permissions` is empty

Because it is true. `capabilities` must be non-empty — a plugin that answers no
questions is not a plugin — but `permissions` may be empty, and a plugin that
asks for `network` "just in case" costs the user a prompt for a facility it
never touches. Over-permissioning is exactly the failure the permission model
exists to make visible, and the reference plugin should not be the first
example of it.

`example/test/permission_guard_test.dart` proves the claim rather than
asserting it in prose: it runs the whole lifecycle plus a search and every
catalog method against a `FakeSwayvePluginContext` with **nothing** granted,
and the fake enforces that set through the same mixin the host uses. If a later
edit adds an HTTP call, that test fails before the change reaches anyone's
phone.

### Why it claims no playback

`media.streamable` and `media.downloadable` are both `false`, every track's
`availability` is `SwayveAvailability.none`, and there is no
`SwayveStreamProvider` anywhere in the package.

This is the point worth taking away: **a plugin may contribute catalogue data
without claiming any playback rights.** Streamable, downloadable and on-device
are three independent facts, and none follows from "this plugin knows about
this track". A metadata source, a lyrics provider or a recommendation engine
has plenty to contribute and nothing to play, and the manifest, the models and
the registered providers must all say so consistently — because the host
believes them. A plugin that overstates `availability` does not get caught by a
validator; it gets caught by a user pressing play on something that cannot
play.

## The eight things the example is trying to teach

1. **The lifecycle.** `load → initialize → registerProviders → active →
   dispose`. The entrypoint function is cheap and synchronous; `initialize`
   registers and returns; `dispose` is idempotent and does not touch the
   network.
2. **Why providers are registered in `initialize`.** It is how the host learns
   what a plugin can do without knowing what a plugin *is*. There is no
   `if (plugin.id == 'example')` anywhere in Swayve, and nowhere for one to go.
3. **Why a plugin never renders UI.** Plugins supply data; the host draws it.
   That is why the SDK is pure Dart with no Flutter dependency — a plugin that
   could inject widgets could break the app's layout, theme, accessibility and
   frame budget, and "a broken plugin must never break Swayve" is the principle
   the whole design is bent around.
4. **Normalization happens at the plugin's boundary.**
   `lib/src/catalogue.dart` holds records shaped the way real music APIs are
   shaped — a credit as one string, a duration as an integer — and converts
   them into SDK models before anything else sees them. That single conversion
   point is what lets the host render every provider's data identically.
5. **`extra` carries provider-specific data the host must not interpret.** The
   host carries it, persists it and hands it back untouched. Use it to save
   yourself a lookup; never to smuggle in something the host is supposed to act
   on, because the moment the host read it, it would have to know which plugin
   wrote it.
6. **Cancellation is checked inside the loop, not only before it.** A
   cooperative provider stops when asked instead of finishing a scan the host
   stopped caring about. `_withinOperationDeadline` in `providers.dart` shows
   the other half — a plugin that notices its own overrun reports
   `SwayvePluginTimeoutException`, a more accurate thing to show a user than
   the `SwayvePluginUnavailableException` the host would otherwise invent.
7. **Failures are `SwayvePluginException` subtypes, never raw throws.** The
   host isolates a raw throw too, but cannot interpret one, so an unknown error
   can only become a generic "temporarily unavailable".
8. **Not-found is a value, not a failure.** `ExampleCatalogProvider.album`
   returns `null` for an id that is not in the catalogue. An empty search
   result means "I searched and there was nothing"; an exception would mean
   "this source is down", and Swayve shows those two very differently.

## Deliberate omissions

Things a real plugin would have that this one does not, so their absence is not
a surprise:

- **No artwork.** No `SwayveArtworkProvider`, every `artwork` field `null` —
  with no network and no bundled assets there is no honest image to point at,
  and a `SwayveImageRef` to a URL that 404s is worse than none.
- **No settings.** No `settings` block, so nothing for the host to render. See
  [`reference/`](reference) for what one looks like.
- **No search cursor.** The whole catalogue fits in one response, so search
  returns `cursor: null`. Browsing *does* page, because that is where the
  cursor contract is worth demonstrating.
- **No `network.hosts`.** It makes no outbound requests, so there is nothing to
  declare and no `network` permission to declare it under.

## Depending on the SDK

A plugin depends on the SDK and on nothing else. Every capability it needs from
the outside world — HTTP, storage, secrets, a web view — arrives through
`SwayvePluginContext`.

```yaml
dependencies:
  swayve_plugin_sdk:
    git:
      url: https://github.com/dazacode/swayve-plugins.git
      path: packages/swayve_plugin_sdk
      ref: 3d1ffa237bfc7c0b3f7acb0651682fc9448e5bbd
```

Pin the `ref` to a commit, never a branch. `ref: main` means a fresh checkout
of your plugin can resolve a different SDK than the one you tested against, and
that failure surfaces as a compile error in someone else's CI. Bump it
deliberately, and re-run the validator when you do.

## For AI agents

If you are an agent generating a Swayve plugin, read [`AGENTS.md`](AGENTS.md)
first. It states the constraints that are not obvious from the schema and that
a plausible-looking generated plugin usually gets wrong.

## Licence

Apache-2.0. See [`LICENSE`](LICENSE). The catalogue in
`example/lib/src/catalogue.dart` is fictional — the artists, albums and tracks
named there were invented for this plugin.
