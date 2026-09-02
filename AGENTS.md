# Notes for AI agents

You are probably here because someone asked you to write a Swayve plugin. This
file is the set of constraints that are *not* obvious from the schema, and that
a plausible-looking generated plugin usually gets wrong.

Read [`example/`](example) before writing anything. It is short and it is the
answer to most structural questions.

## Verify, do not assert

There is a validator. Run it. A plugin that "looks right" is worth nothing next
to one that passes:

```bash
git clone https://github.com/dazacode/swayve-plugins.git
cd swayve-plugins && dart pub get
dart run tools/validate_plugin.dart ../path/to/my_plugin --strict
```

`--strict` turns warnings into errors, which is what CI does. Then, from inside
the plugin:

```bash
dart pub get && dart analyze && dart test
```

Do not report a plugin as finished until all four of those pass. If you cannot
run them, say so plainly rather than claiming the plugin works.

## The rules that are easy to get wrong

1. **`entrypoint` must equal the plugin's directory name**, and it must match
   `^[a-z][a-z0-9_]*$`. No hyphens, ever. Renaming the directory without
   renaming `entrypoint` is the single most common first failure.

2. **`id` is reverse-DNS with at least three segments** —
   `dev.yourname.swayve.my_plugin`. It is not the entrypoint, and it is not the
   `sourceId`. Three different identifiers, three different jobs.

3. **`permissions` may be empty, and empty is a good answer.** Do not declare
   `network` because a music plugin "probably needs it". Declare what the code
   actually uses. Every unnecessary permission is a prompt shown to a real
   person for a facility that is never touched.

4. **A plugin depends on the SDK and on nothing else.** No `http`, no `dio`, no
   `shared_preferences`, no `path_provider`. HTTP, storage, secrets and web
   views all arrive through `SwayvePluginContext`. A direct transport
   dependency bypasses the permission model entirely, which is the reason the
   rule exists. If you find yourself adding a package to `dependencies`, stop.

5. **A plugin never imports Flutter and never renders UI.** The SDK is pure
   Dart. Plugins supply data; the host draws it. This includes settings — you
   declare them in the manifest and the host renders them.

6. **Do not derive one media fact from another.** `streamable`, `downloadable`
   and `offlineCache` are independent. Knowing about a track is not a claim
   that you can play it.

7. **Not-found is a value, not an exception.** Return `null` for an id you do
   not have; return an empty page for a search with no matches. Throw only when
   the *source* failed. Swayve shows those two things very differently, and it
   believes you.

8. **Throw `SwayvePluginException` subtypes, never raw errors.** The host
   isolates a raw throw, but cannot interpret it, so it degrades to a generic
   "temporarily unavailable".

9. **Check the cancellation token inside loops, not just before them.** A
   provider that finishes a scan the host stopped caring about is wasting a
   user's battery.

10. **Pin the SDK `ref` to a commit, never a branch.** `ref: main` makes builds
    non-reproducible and the failure lands in someone else's CI.

## Do not invent vocabulary

`capabilities`, `permissions`, `platforms`, `runtime`, `settings[].type`,
`source.contentTypes` and `source.availability` are **closed enums**. The full
lists are in the [README](README.md#the-closed-vocabularies), and the schema is
authoritative:

```
https://github.com/dazacode/swayve-plugins/blob/main/schema/swayve-plugin.schema.json
```

A value outside those sets fails validation. Read the enum; do not guess a
plausible-sounding one.

## Do not copy the fixture data

`example/lib/src/catalogue.dart` is invented data whose only job is to make the
example runnable offline. Replace it with a real client. Keep its *shape* —
raw data in, SDK models out, converted at one boundary — because that is the
part worth copying.

## Writing the manifest

Start from `example/plugin.json` (fourteen required fields plus three cheap
optionals) rather than from
[`reference/plugin.reference.json`](reference/plugin.reference.json), which
declares everything and is a lookup table, not a starting point. Add optional
fields when the plugin actually needs them.

Write new manifests at `schemaVersion: 7`. Versions 1 through 6 still validate,
so a generated `1` will pass and still be wrong.
