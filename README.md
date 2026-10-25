# F1RaceEngineer-content

The public content feed for [F1 Race Engineer](https://github.com/TrulyAsleep/F1RaceEngineer-dist):
voice packs, strat packs, data packs and the news the app shows on its home page - later
themes, dashes and plugins. The app reads this repo; nothing here is code the app runs.

## What is here

| file | what it is |
| --- | --- |
| `v2/catalog.json` | the list of downloadable items and the news feed that 0.5.0-beta.2 and later read, human-edited, plain JSON (schema 2) |
| `v2/catalog.sig` | its detached signature (made by the release tool with the CONTENT key, never edited by hand) |
| `catalog.json` | the same for 0.4.x and 0.5.0-beta.1, which read only schema 1: voice packs and news (see Two catalogs) |
| `catalog.sig` | its detached signature |
| `scripts/publish.md` | the exact command sequence for adding or updating an item |
| Releases | every item is a zip attached to a release of this repo, one release per item version (tag `<kind>-<id>-<n>`, for example `voice-kokoro-en-1`) |

## How the app reads it

1. The app fetches `catalog.json` and `catalog.sig` from the raw files on `main`
   (`https://raw.githubusercontent.com/TrulyAsleep/F1RaceEngineer-content/main/v2/...` from
   0.5.0-beta.2 on, the root before that), with `If-None-Match` against the ETag it cached
   last time, so an unchanged catalog costs a 304.
2. It verifies the signature with the content public key embedded in the build. A catalog
   that does not verify is never shown - the last verified copy (cached under the app data
   folder, or the copy bundled into the build) stands in, with the reason in the log.
3. It validates the catalog: every item has a known kind, a safe id, a version, a size, a
   SHA-256, and a URL on `github.com/TrulyAsleep/` or `raw.githubusercontent.com/TrulyAsleep/`
   over https. Anything else fails the whole catalog.
4. A download is streamed to `content\<kind>\<id>.download.part` (resumed with a Range
   request after an interruption), hashed as it arrives, and compared with the catalog's
   SHA-256 BEFORE the zip is opened.
5. Inside every item zip is `item.trulyasleep`: a manifest signed with the same content
   key that names the item (`content:<kind>:<id>`), its version, and the SHA-256 of every
   file in the zip. The app verifies it, extracts into a `.partial` folder, verifies the
   folder again, and only then swaps it into place. The item verifies offline too, so a
   zip copied to another PC can be imported from a file.
6. Before an installed item is read (a voice pack opened, a strat pack loaded) the folder
   is verified against its manifest once more. A folder that has been edited is refused.

Two signing keys exist on purpose. The RELEASE key signs the app itself (release
manifests and update zips). The CONTENT key signs this catalog and every item. The app
never lets one vouch for the other: a new pack needs no app release, and a leaked content
key cannot forge an app update. Full details: `docs/CONTENT-CATALOG.md` in the app repo.

## Two catalogs

| path | schema | read by | lists |
| --- | --- | --- | --- |
| `v2/catalog.json` + `v2/catalog.sig` | 2 | 0.5.0-beta.2 and later | everything: voice packs, language packs, any later kind |
| `catalog.json` + `catalog.sig` | 1 | 0.4.x and 0.5.0-beta.1 | voice packs and the news |

A 0.4.x build refuses the whole catalog for any schema stamp but 1, and for a single item
of a kind it does not know. So the first language packs in the root catalog (22 Oct 2026)
left a fresh 0.4.5 install, which has no cached copy to fall back on, with nothing to
download - the voice pack included. The root is the 9 Sep 2026 catalog again, byte for
byte with its own signature, and the newer builds read `v2/`.

- Everything goes into `v2/`. A voice pack goes into the root as well; a language pack
  or any later kind never does (`content add` would stamp the root schema 2 and every
  0.4.x build would stop reading it). Sign and verify each catalog you change.
- News for 0.4.x drivers goes into the root, news for 0.5 and later into `v2/`.
- 0.5.0-beta.1 reads the root. It keeps its cached catalog when the one served is older,
  so a beta.1 that already fetched the 22 Oct catalog keeps its language list. Re-signing
  the root with an `updated` date after 2026-10-22 replaces that copy, and beta.1 then
  lists no languages.
- 0.5.0-beta.2 and later ask for `v2/` first and read the root only when `v2/` answers 404.

## Publishing an item

See `scripts/publish.md`. In short: pack the folder with the release tool (it writes the
signed `item.trulyasleep` inside the zip and prints the size and SHA-256), create a release
on this repo and attach the zip, add the entry to `catalog.json` with `content add`, sign
the catalog with `content sign`, check everything with `content verify`, commit and push
`catalog.json` and `catalog.sig` together. The app picks the change up on its next fetch.

## Items

| item | version | release | note |
| --- | --- | --- | --- |
| `voice:kokoro-en` | 1.0.0 | `voice-kokoro-en-1` | the Kokoro neural voice pack (English, 53 speakers), needs app 0.4.1-beta.1 or later; in both catalogs |
| `lang:fr`, `lang:de`, `lang:es`, `lang:it`, `lang:pt-br`, `lang:tr` | 1.0.0 | `lang-<code>-1` | the six language packs, draft tier, need app 0.5.0-beta.1 or later; in `v2/` only |

The pack published on the app's dist repo under `voice-packs` is the 0.4.0 beta's copy of the
same model without the signed `item.trulyasleep` inside. 0.4.0 keeps using that one; 0.4.1
and later read this catalog and fetch the signed pack from this repo.

## Licences

Model files inside a voice pack keep their own licence file (the Kokoro pack ships
Apache 2.0).
