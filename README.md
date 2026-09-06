# F1RaceEngineer-content

The public content feed for [F1 Race Engineer](https://github.com/TrulyAsleep/F1RaceEngineer-dist):
voice packs, strat packs, data packs and the news the app shows on its home page - later
themes, dashes and plugins. The app reads this repo; nothing here is code the app runs.

## What is here

| file | what it is |
| --- | --- |
| `catalog.json` | the list of downloadable items and the news feed, human-edited, plain JSON |
| `catalog.sig` | its detached signature (made by the release tool with the CONTENT key, never edited by hand) |
| `scripts/publish.md` | the exact command sequence for adding or updating an item |
| Releases | every item is a zip attached to a release of this repo, one release per item version (tag `<kind>-<id>-<n>`, for example `voice-kokoro-en-1`) |

## How the app reads it

1. The app fetches `catalog.json` and `catalog.sig` from the raw files on `main`
   (`https://raw.githubusercontent.com/TrulyAsleep/F1RaceEngineer-content/main/...`), with
   `If-None-Match` against the ETag it cached last time, so an unchanged catalog costs a 304.
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

## Publishing an item

See `scripts/publish.md`. In short: pack the folder with the release tool (it writes the
signed `item.trulyasleep` inside the zip and prints the size and SHA-256), create a release
on this repo and attach the zip, add the entry to `catalog.json` with `content add`, sign
the catalog with `content sign`, check everything with `content verify`, commit and push
`catalog.json` and `catalog.sig` together. The app picks the change up on its next fetch.

## Items

| item | version | release | note |
| --- | --- | --- | --- |
| `voice:kokoro-en` | 1.0.0 | `voice-kokoro-en-1` | the Kokoro neural voice pack (English, 53 speakers), needs app 0.4.1-beta.1 or later |

The pack published on the app's dist repo under `voice-packs` is the 0.4.0 beta's copy of the
same model without the signed `item.trulyasleep` inside. 0.4.0 keeps using that one; 0.4.1
and later read this catalog and fetch the signed pack from this repo.

## Licences

Model files inside a voice pack keep their own licence file (the Kokoro pack ships
Apache 2.0).
