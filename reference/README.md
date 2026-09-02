# Manifest reference

[`plugin.reference.json`](plugin.reference.json) is a single manifest that
declares **every field the schema defines** — all fourteen required, all ten
optional — in a document that validates against the real schema.

It is not a working plugin. There is no `lib/`, no tests, and the plugin it
describes does not exist. It is here to be looked things up in.

## How to use it

Start your own manifest from [`../example/plugin.json`](../example/plugin.json),
which is the required fields plus three cheap optionals. When you need a field
you have not used before, find it here to see its shape, then add just that
field.

Copying this file wholesale is the wrong move: it declares nine capabilities,
four permissions and a session capture flow, and a plugin should declare only
what it implements.

## Checking your own manifest

Against the schema alone:

```bash
pip install check-jsonschema
curl -sSLO https://raw.githubusercontent.com/dazacode/swayve-plugins/main/schema/swayve-plugin.schema.json
check-jsonschema --schemafile swayve-plugin.schema.json my_plugin/plugin.json
```

Against the schema *and* the rules the schema cannot express — that
`entrypoint` matches the directory name, that a `session_capture` target names
a real `secret` setting, that declared capabilities line up with registered
providers:

```bash
git clone https://github.com/dazacode/swayve-plugins.git
cd swayve-plugins && dart pub get
dart run tools/validate_plugin.dart ../my_plugin --strict
```

Use the second one. The first is a subset.

## Field notes

A few fields whose purpose is not obvious from their type.

**`source` is not `capabilities`.** `capabilities` is what interfaces you
implement; `source` is whether you should appear to a person as a *place music
comes from*, and what they may ask it for. A service built on user uploads can
implement `catalog` honestly and still have no albums worth querying, so
`source.contentTypes` is declared rather than derived.

**`sourceId` is not `id`.** `id` identifies a package. `sourceId` identifies
the place the music comes from, and is what a saved source selection is keyed
on. Two plugins could in principle serve one service.

**`iconName` is not a path.** It is a glyph name the host resolves from its own
icon set. An unknown name falls back to a generic mark. `icon`, separately, *is*
a path, and must live under `assets/`.

**`session_capture.capture[].as_secret` must name a real setting.** It has to
match the `id` of a `settings` entry with `type: secret` — that is where the
captured value is written. The reference manifest shows the pair.

**`timeouts` is a request, not a guarantee.** The host applies its own hard
limits regardless.

**`network.hosts` is a declaration, not a firewall you configure.** It is what
the host enforces your outbound requests against. Requesting a host you never
use is over-permissioning in the same way an unused `permission` is.
