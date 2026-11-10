# Deployment

Gemstorm 3D: Crystal Cascade is a static Unity WebGL build. There is no bundler, no
package manager and no server code: `scripts/prepare-site.mjs` reshapes the export and
GitHub Actions publishes the result.

## Where it lives

| Piece | Value |
| --- | --- |
| Repository | `jd4rider/gemstorm-3d-web` (public) |
| Play URL | <https://jd4rider.github.io/gemstorm-3d-web/play/> |
| Cover page | <https://jd4rider.github.io/gemstorm-3d-web/> |
| Host | GitHub Pages, Actions source (`pages.yml`), no custom domain |
| Catalogue | `littlejoyarcade/src/data/games.ts` + `supabase/migrations/0018_gemstorm_3d.sql` |
| Portal entry | <https://littlejoyarcade.site/#/game/gemstorm-3d> |
| Game id | `gemstorm-3d` |

The portal embeds the `play/` URL in an iframe, so the game has to keep working inside a
nested browsing context: no custom domain, no service worker, no `X-Frame-Options`.

## Why no custom domain

GitHub gives a Pages site exactly one canonical hostname. A `CNAME` file or a Pages custom
domain would make `jd4rider.github.io/gemstorm-3d-web/` answer with a 301 to a hostname
whose certificate may not exist yet, and the arcade's embedded player would show a black
screen. Game repos therefore stay on `github.io`; the portal links that URL directly. See
`../littlejoyarcade/DEPLOYMENT.md`.

## Deploy

```bash
git push origin main
gh run list --repo jd4rider/gemstorm-3d-web --limit 3
```

The workflow runs `node scripts/prepare-site.mjs`, asserts the payload is servable (wasm
magic bytes, no `.br` references, every required file present), asserts the arcade account
layer survived, uploads `site/` and deploys it. Pages must have its source set to **GitHub
Actions** once:

```bash
gh api -X POST repos/jd4rider/gemstorm-3d-web/pages -f build_type=workflow
```

## Verify a deploy

```bash
curl -sI https://jd4rider.github.io/gemstorm-3d-web/play/                     # 200 text/html
curl -sI https://jd4rider.github.io/gemstorm-3d-web/play/Build/WebGL.wasm
curl -s -r 0-3 https://jd4rider.github.io/gemstorm-3d-web/play/Build/WebGL.wasm | xxd -p
# expect 0061736d (\0asm) - a Brotli or gzip payload would start with something else
```

A 200 is not proof the game runs: the payload is 58 MB, so also open the URL in a browser
and wait for the progress bar to disappear. The arcade clients log
`[LittleJoy] Running on local fallback (SDK not present).` to the console on a direct
visit, which is expected — the SDK only initialises when the page is loaded with
`?game_id=gemstorm-3d` from the portal.

## What "the account layer survived" means

The Unity export regenerates `index.html` from its template on every build. If that file
replaces the prepared one, the game still boots — it just silently stops signing players in
and stops saving to the cloud. Three things guard against that:

1. `prepare-site.mjs` copies `public/arcade-unity.js` and `public/arcade-client.js` next to
   the build and injects their two `<script>` tags before `</body>`, shim first.
2. The same script throws if either file or either tag is missing, if the tags are in the
   wrong order, if the fill-the-window patch did not apply, or if the account pill's
   offset that clears the Unity footer is gone.
3. The `Check the arcade account layer survived` CI step re-asserts all of it from the
   generated `site/`, and refuses to ship anything containing `service_role`.

`arcade-client.js` is the platform's canonical file and must stay byte-identical. The pill
it renders is pinned with an inline style, so the page raises it over the Unity footer with
an `!important` rule in the injected CSS rather than by editing the client.

## Refresh the payload

The Unity project lives separately (`~/Developer/Gemstorm 3D`). Its build script
`Assets/_Gemstorm/Editor/BuildWebGL.cs` writes straight to `Deploy/build/WebGL/`:

```bash
/Applications/Unity/Hub/Editor/7000.0.0a12/Unity.app/Contents/MacOS/Unity \
  -batchmode -projectPath "/Users/jonathanforrider/Developer/Gemstorm 3D" \
  -executeMethod Gemstorm.EditorTools.BuildWebGL.Build -logFile /tmp/gemstorm-build.log
```

Then copy `Deploy/build/WebGL/` here and commit. Do **not** copy the prepared
`Deploy/site/` — CI regenerates it.

## Payload notes

- 58 MB total, most of it `WebGL.wasm`. GitHub Pages allows 1 GB per site and 100 MB per
  file, so the build fits, but the first load is slow on mobile.
- Pages gzips the wasm in transit; the payload is stored uncompressed because a Brotli
  export would need `Content-Encoding: br`, which Pages cannot send.
- Unity's loader asks for the data and wasm files with `must-revalidate`, so repeat visits
  do not re-download the payload once the browser has it cached.

## Publishing to the catalogue

The portal reads its catalogue from the `public.games` table, falling back to
`src/data/games.ts`. A new game updates both:

```bash
cd ../littlejoyarcade
# 1. add the entry to src/data/games.ts
# 2. add supabase/migrations/0018_gemstorm_3d.sql (same shape as 0010_cannon_clash_3d.sql)
npx supabase link --project-ref fqovramltegcfiyebyrt
npx supabase db push --remote --include-all --yes
git add -A && git commit -m "Add Gemstorm 3D to the catalogue" && git push
```

`db push` is what makes the row live; the push to `main` is what rebuilds the portal site
itself. The `public.games` row must exist before cloud save can land, because
`game_progress.game_id` has a foreign key onto it.

## Rollback

Pages keeps the last deploy only. To roll back, revert the commit that changed
`build/WebGL/` and push:

```bash
git revert <sha> && git push
```
