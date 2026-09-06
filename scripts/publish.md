# Publishing an item

Everything runs from the APP repo (`F1RaceEngineer`), where the release tool lives; this
content repo is checked out beside it. `$content` below is the path to this repo.

```pwsh
$tool = 'dotnet run --project tools/F1RaceEngineer.ReleaseTool -c Release --'
$content = '..\F1RaceEngineer-content'
```

## Once: the content key

```pwsh
Invoke-Expression "$tool new-key --key content"          # writes content-key.dat (DPAPI) and the first line of src\F1RaceEngineer.Core\Integrity\content.pub
Invoke-Expression "$tool export-key --key content $env:USERPROFILE\Desktop\content-key.pem"   # back it up in the password manager, then delete the file
dotnet build F1RaceEngineer.sln -c Release               # Core now embeds the key; ship an app release carrying it before any content is published
```

## Every item

1. Pack the folder. For the Kokoro voice pack:

   ```pwsh
   Invoke-Expression "$tool voice-pack dist\voices\kokoro-int8-multi-lang-v1_0 --out dist\voices\F1RaceEngineer-Voice-Kokoro-EN.zip"
   ```

   For anything else (a strat pack, a data pack):

   ```pwsh
   Invoke-Expression "$tool content pack <folder> --kind strat --id spa-2026 --version 1.0.0 --out dist\content\F1RaceEngineer-Strat-spa-2026-1.0.0.zip"
   ```

   The tool writes a signed `item.trulyasleep` at the zip root, verifies the zip against
   it, and prints the size, the SHA-256 and a ready-to-paste catalog entry.

2. Create a release on this repo: tag `<kind>-<id>-<n>` (`voice-kokoro-en-1`,
   `strat-spa-2026-1`), not a draft. Attach the zip, name unchanged. Copy the asset URL
   (`https://github.com/TrulyAsleep/F1RaceEngineer-content/releases/download/<tag>/<file>`).

3. Add or replace the catalog entry (size, hash, version and kind:id come from the zip's
   own manifest; the text fields are yours, and an existing entry's text is kept when a
   flag is left out):

   ```pwsh
   Invoke-Expression "$tool content add $content\catalog.json --from-pack dist\voices\F1RaceEngineer-Voice-Kokoro-EN.zip --url https://github.com/TrulyAsleep/F1RaceEngineer-content/releases/download/voice-kokoro-en-1/F1RaceEngineer-Voice-Kokoro-EN.zip --name 'Neural voice (Kokoro, English)' --summary 'More natural engineer voice that runs on your CPU - 12 English speakers, about 120 MB'"
   ```

   Or edit `catalog.json` by hand and paste the printed entry; `content add <catalog.json> <entry.json>` inserts a file.

4. News, when there is any:

   ```pwsh
   Invoke-Expression "$tool content news add $content\catalog.json --date '06 SEP 2026' --title 'Downloads' --body 'Voice packs and more, from the new Downloads page.'"
   ```

5. Sign and check:

   ```pwsh
   Invoke-Expression "$tool content sign $content\catalog.json"
   Invoke-Expression "$tool content verify $content"        # catalog + signature + every referenced zip found in $content or $content\packs
   ```

   `content verify` exits 0 only when the signature verifies with the key embedded in the
   current build of Core, the catalog validates, and every zip present locally matches
   its entry's size and hash and its own manifest.

6. Commit and push `catalog.json` AND `catalog.sig` in the same commit. The raw files are
   served by GitHub's CDN with a few minutes of caching; a client that sees the new json
   with the old sig refuses the pair and keeps its last verified copy until both agree.

## Updating an item

Same steps with a new version (`--version 1.1.0`) and a new release tag
(`strat-spa-2026-2`). The app compares versions with semver precedence and offers the
update; the old release can stay for the record.

## Removing an item

Delete its entry from `catalog.json`, `content sign`, `content verify`, commit. Installed
copies stay installed (the app never deletes content the driver chose); the item is no
longer offered.
