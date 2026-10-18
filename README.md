# coho-sdk-spec

The shared material every Coho SDK is built and tested against. Each language's SDK copies what it needs from a tagged release of this repository, so all of them agree with the server and with each other.

```
type-model/
├── type-model.schema.json   JSON Schema for the type model
└── fixtures/
    ├── cli-next-blog.json   the type model of the cli-next-blog sample
    ├── delivery-fixture.json   the type model of the delivery recordings' content
    └── assets.json          asset fields, with number, json and a repeater
delivery/
└── recordings/              what the delivery API answered to a fixed set of requests
```

Nothing here is edited by hand. Each file is produced by the Coho server's own code and copied here, so a fixture is exactly what a generator will receive.

## The type model

A language-neutral description of one branch's or tag's content types. Code generators read it and write typed code: one type per content type, with the right fields, optionality, and reference targets. The server has already applied Coho's rules, so a generator never works them out for itself.

### Rules for generators

- **Versioning.** `irVersion` is `major.minor`. Ignore keys you do not know. Refuse a major version you do not support, and name both versions in the error. A minor version only adds optional keys or field kinds.
- **Optionality comes from `presence` alone.** `always` means delivery always includes the field; `maybe` means it can be absent. A localized field is `maybe` even when required, because delivery leaves it out wherever the requested locale has no translation.
- **Choice values are open.** Content can hold a value added after the file was produced. Generated code must accept an unknown value and keep it, not fail: in TypeScript, `'a' | 'b' | (string & {})`; elsewhere, an explicit unknown member carrying the raw value.
- **References name types in the same file.** Every slug in a reference field's `of` is a key of `types`. An empty `of` means the link points at an entry whose type this branch or tag does not have; generate it as an untyped entry.
- **Asset fields** (`kind: asset`, since 1.1) hold files: delivery embeds each asset as an object (`id`, `title`, `alt`, `focalPoint`, `file`, `url`, and for raster images `imageUrl`), never as a link to follow. `many` makes it a list. `accept` lists the media types a linked file may be, or family wildcards such as `image/*`, and is empty for any. **A single asset field is nullable whatever its `presence`**: an asset deleted on the branch or tag is delivered as `null`. In a list, a deleted asset is left out, never `null`. A field whose `accept` names only raster images (`image/png`, `image/jpeg`, `image/gif`, `image/webp`, `image/avif`, `image/heic`) may be typed as an image that always has dimensions; `image/*` includes SVG, which has none.
- **Unknown field kinds are untyped.** If `kind` is one you do not know, emit the field as untyped and print a warning. Never fail.
- **Print `warnings`.** When present, each one names a type and field that need attention.
- **Fields hidden from the API are already gone.** Do not look for them.

### File form

The type model is two-space-indented JSON with `"key": value`, one array element per line, `[]` and `{}` when empty, and a trailing newline: the layout of `JSON.stringify(model, null, 2)`. Tooling that writes the file in that layout produces no diff.

`source.schemaHash` is the SHA-256 of `types`, serialized compactly in the order given. Equal hashes mean equal types, whichever branch or tag they came from.

## Delivery recordings

Each file is one request to the delivery API and its answer, recorded from the server running against a fixed set of content. A delivery SDK replays them as its HTTP layer, so its tests check it against what the server actually sends.

```json
{
  "request":  { "method": "GET", "path": "/v0.0.1/entries-by-slug/chain-1?depth=2&locale=en-US", "auth": "key" },
  "response": { "status": 200, "headers": { "content-type": "…", "cache-control": "…", "etag": true }, "body": { … } }
}
```

- `path` is relative to the project's base URL, `…/v1/projects/{project}`. Compare the query as a set: an SDK may send the same parameters in another order.
- `auth` is `key` for the project's delivery key, `none` for no `Authorization` header, and `other-key` for a valid key that grants a different project.
- `etag` says whether the response carried one. The value is not recorded, because it changes every time the content is rebuilt; a replaying SDK can make up its own.
- The content is a type, `article`, with a localized `title`, a `body`, an integer `rank`, and a `related` reference. Three entries, `chain-1` to `chain-3`, link in a loop, and `chain-2` also links to an entry that does not exist. Four more, `entry-1` to `entry-3` and `entry-fr`, carry translated titles. A second type, `notice`, declares an integer `priority` and a text `message`, and has no entries. A third, `figure`, has asset fields: `image` (images only), `poster` and a `gallery` list. `figure-1` links a JPEG with a focal point, an SVG, a PDF, and an asset that does not exist, in `poster` and in `gallery`.

| Recording | What it shows |
|---|---|
| `entry-by-slug`, `entry-by-slug-depth-1`, `entry-by-slug-depth-2` | One entry's references at depths 0, 1 and 2: `depth` at the boundary, `absent` for the missing entry, and the loop back to the root |
| `entry-by-id-de` | A read by id, projected to `de-DE` |
| `entries-by-id` | A read of several ids, one of which does not exist |
| `entries-ordered` | A page ordered by `-rank`, with more to come |
| `entries-fr` | Every entry projected to `fr-FR`, where most titles are untranslated and so absent |
| `entries-empty-type` | A type with no entries, ordered by its declared field: an empty page, not a refusal |
| `entry-with-assets` | Assets embedded (delivery 2.1.0): an image with `imageUrl`, an SVG and a PDF without one, a missing asset as `null` in `poster` and left out of `gallery`, and the `warnings` that name it. The project id in URLs is `{project}` |
| `entry-not-found`, `ref-not-found`, `invalid-query`, `key-required`, `project-not-found` | The refusals |

## Type model fixtures

| Fixture | Covers |
|---|---|
| `cli-next-blog` | text, longText (plain and markdown), choice, multichoice, datetime, integer, boolean, localized fields, single and many references, and a type that references itself |
| `delivery-fixture` | The types of the delivery recordings' content, `article` and `notice`, so a generator's output can be run against the recordings |

Still to come: a fixture with `number`, `json` and a repeater.

## License

Apache-2.0
