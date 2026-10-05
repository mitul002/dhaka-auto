# Dhaka Auto: Deployment Guideline

This guide is written for an AI coding agent (for example Antigravity) that deploys the Dhaka Auto web game from the unzipped `dhaka-auto` folder. A human can follow it too.

## 0. Rules for the agent

1. **Do not change game logic.** Only touch the items listed in section 6 (placeholders) and section 9 (cache version).
2. **Keep every file at the root of the deploy folder**, with no subfolders. `index.html` must keep that exact name, because `sw.js` caches `./index.html`.
3. Never invent results. If a check cannot be run (for example no browser available), say so in the final report.
4. Ask the human before doing anything that costs money, buys a domain, or changes account billing.

## 1. What is in the package

| File | Purpose |
|---|---|
| `index.html` | The whole game (HTML, CSS and JS in one file). Loads Three.js r128 from cdnjs and fonts from Google Fonts. |
| `manifest.webmanifest` | PWA manifest (app name, icons, fullscreen display). |
| `sw.js` | Service worker for offline play. Has a cache version string `CACHE`. |
| `icon-192.png`, `icon-512.png`, `icon-maskable-512.png` | App icons. |
| `og-image.png` | Link-share preview image (1200x630). |
| `vercel.json` | Vercel headers (`sw.js` never cached, correct manifest type). |

The main game script is stored inside `<script type="text/plain" id="gameSrc">` and started by a small loader script. That is intentional (it drives the real loading bar). Do not "fix" it.

## 2. Prerequisites

- Node.js 18 or newer (`node -v`)
- A Vercel account (free Hobby plan works for personal, non-commercial use)
- Vercel CLI: `npm i -g vercel`
- Optional: Git and a GitHub account if deploying through a repository

**Plan note for the human:** Vercel Hobby is restricted to non-commercial personal use. If the game will show ads, take payments or otherwise earn money, use a Pro plan, or host on Netlify or Cloudflare Pages (see section 11).

## 3. Pre-deploy checks (run before deploying)

Run these in the unzipped folder.

```bash
# 3.1 all files present, none in subfolders
ls -1
# expected: icon-192.png icon-512.png icon-maskable-512.png index.html
#           manifest.webmanifest og-image.png sw.js vercel.json

# 3.2 JSON files are valid
node -e "JSON.parse(require('fs').readFileSync('manifest.webmanifest','utf8')); JSON.parse(require('fs').readFileSync('vercel.json','utf8')); console.log('json ok')"

# 3.3 service worker syntax
node --check sw.js && echo "sw ok"

# 3.4 game script syntax (extracts the biggest script block from index.html)
python3 - <<'PY'
import re
s = open('index.html', encoding='utf-8').read()
js = max(re.findall(r'<script[^>]*>(.*?)</script>', s, re.S), key=len)
open('/tmp/game_check.js', 'w', encoding='utf-8').write(js)
PY
node --check /tmp/game_check.js && echo "game script ok"
```

All four must pass. If one fails, stop and report the exact error. Do not edit game code to work around it.

## 4. Test locally first

Service workers only run on `https` or `localhost`, so test on localhost.

```bash
# pick one
npx serve . -l 8080
# or
python3 -m http.server 8080
```

Open `http://localhost:8080` in Chrome and check:

1. A loading screen appears. The bar moves through three stages with the messages "Loading game engine", "Loading fonts", "Building the city" (Bangla by default).
2. The loader disappears and the start screen shows: title, "Today's goals" card (3 goals in one row), and "Choose game mode". There must be **no** "How to play" legend on the start screen.
3. Bottom-left: Settings and Achievements buttons. Bottom-right: Shop. Top-left: Info.
4. Start a game in any mode. Controls work: arrow keys, Space (jump), N or Shift (nitro, only when the bar is full).
5. Open DevTools, then Console. Report any red errors. Warnings about fonts are fine.
6. DevTools, Application, Manifest: no errors. Service Workers: `sw.js` is activated.

If the loader shows a failure message, the text under it in square brackets is the real reason. Copy it into the report.

## 5. Deploy to Vercel (CLI)

```bash
cd dhaka-auto            # the folder that contains index.html
vercel login             # first time only
vercel whoami            # confirm login
vercel                   # creates a preview deployment, answer the prompts below
```

Answer the prompts like this:

| Prompt | Answer |
|---|---|
| Set up and deploy? | Yes |
| Which scope? | the human's personal account |
| Link to existing project? | No |
| Project name | `dhaka-auto` (or what the human wants) |
| In which directory is your code located? | `./` |
| Framework / build settings | **Other**. Leave Build Command, Output Directory and Install Command **empty** (this is a static site, nothing to build). |

When the preview URL works, publish to production:

```bash
vercel --prod
```

Save the production URL, for example `https://dhaka-auto.vercel.app`. It is needed in section 6.

### Alternative: GitHub import

1. Create a GitHub repo and push all 8 files at the repo root.
2. In Vercel: Add New, Project, import the repo.
3. Framework Preset: **Other**. Build Command and Output Directory: empty. Deploy.

## 6. Post-deploy edits (the only allowed edits in `index.html`)

Social-share crawlers (WhatsApp, Facebook, Twitter) need an **absolute** image URL. Replace the relative `og-image.png` in two meta tags and add `og:url`.

Let `DOMAIN` be the production domain without `https://`, for example `dhaka-auto.vercel.app`.

1. In `index.html`, find these two tags:
   ```html
   <meta property="og:image" content="og-image.png">
   <meta name="twitter:image" content="og-image.png">
   ```
   Change `og-image.png` to `https://DOMAIN/og-image.png` in both.
2. Add this line next to the other `og:` tags:
   ```html
   <meta property="og:url" content="https://DOMAIN/">
   ```
3. Redeploy: `vercel --prod`

(Do the edit with your editor tools. If using `sed`, remember that macOS needs `sed -i ''` and Linux needs `sed -i`.)

## 7. Production verification checklist

Open the production URL and verify each item. Mark each pass or fail in the report.

- [ ] Page loads over `https`, loading bar completes, game starts.
- [ ] No red errors in the console.
- [ ] DevTools, Application, Manifest shows the name "Dhaka Auto" and 3 icons, with no errors.
- [ ] DevTools, Application, Service Workers shows `sw.js` as activated and running.
- [ ] Offline test: after one full load, set Network to Offline, reload. The game must still open. (If cdnjs or fonts were never cached, the first offline load can fail. Load once online first.)
- [ ] Settings (bottom-left gear): Graphics Low/Medium/High, volume sliders, text size, high contrast, colour-blind, Backup export/import, About & Privacy all open and work.
- [ ] Share preview: paste the URL into https://developers.facebook.com/tools/debug/ and into a WhatsApp chat. The image and text should appear. Previews are cached, so use the debugger's refresh button if it shows the old state.
- [ ] Phone test (Android Chrome): game fits the screen, touch controls work, an "Install app" row appears in Settings or the browser shows "Install". iPhone: Safari, Share, Add to Home Screen (no install button on iOS).
- [ ] Optional: Lighthouse, PWA and Performance audit. Report scores honestly.

## 8. Known limits (state these in the report, do not hide them)

- The game's author tested code and syntax only. It has not been fully play-tested in a real browser on many devices, so layout or gameplay bugs are possible.
- Three.js and Google Fonts load from public CDNs. If cdnjs is blocked or down and nothing is cached, the loader shows an error with a Try again button.
- All progress (coins, upgrades, best scores) lives in the browser's `localStorage`. Clearing site data erases it. Players can use Settings, Backup to export and import a code.
- Analytics are **off**. `ANALYTICS_URL` in `index.html` is empty, so events stay on the device. If anyone sets it, the "About & Privacy" text must be updated, because it currently says there is no tracking.
- The weekly challenge gives everyone the same obstacle sequence for the week, but row spacing still depends on player speed.

## 9. Releasing an update later

1. Edit `index.html` (or other files).
2. In `sw.js`, bump the version: change `const CACHE = 'dhaka-auto-v1';` to `v2`, then `v3`, and so on. Without this, phones keep serving the old cached game.
3. Re-run section 3 checks.
4. `vercel --prod`
5. On a phone, open the site twice (the first visit installs the new service worker, the second shows the new version).

Roll back: Vercel dashboard, Deployments, pick a previous deployment, Promote to Production.

## 10. Custom domain (optional, ask the human first)

1. Vercel dashboard, Project, Settings, Domains, Add.
2. Add the DNS records Vercel shows at the domain registrar.
3. After it is live, repeat section 6 with the new domain and redeploy.

## 11. Other hosts (static, no build)

The same 8 files work on any static host. Publish directory is the folder itself.

- **Netlify:** drag the folder to https://app.netlify.com/drop (no commercial-use restriction on the free tier at the time of writing; check current terms). Netlify ignores `vercel.json`; to keep `sw.js` fresh, add a `_headers` file with `/sw.js` and `  Cache-Control: no-cache` on the next line.
- **Cloudflare Pages:** upload the folder in the dashboard (Direct Upload) or connect a Git repo. Build command empty, output directory `/`.
- **GitHub Pages:** works, but the site lives under `/repo-name/`. Use that full path in the section 6 URLs. It needs a public repo on free accounts.

## 12. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Loader shows a failure message with `[...]` | Read the bracket text. "could not download three.min.js" means the CDN is unreachable. Any other text is a script error, so send it with the console output. |
| Old version still shows after deploy | `CACHE` in `sw.js` was not bumped, or the page was opened only once. Bump, redeploy, then open the site twice. |
| No "Install app" row in Settings | Needs https, a valid manifest, and the app not already installed. Chrome desktop and Android only. iOS uses Add to Home Screen. |
| Share preview shows no image | `og:image` is still relative, or the platform cached the old page. Fix the URL, redeploy, then use the Facebook debugger's refresh. |
| No sound at start | Browsers block audio until the first tap or keypress. Not a bug. |
| 404 for `sw.js` or icons | Files were uploaded into a subfolder. Move them to the deploy root. |
| `vercel` asks for a build command | Choose Framework "Other" and leave build settings empty. |

## 13. Final report the agent must produce

Reply with:

1. Production URL.
2. Result of section 3 checks (pass or fail, with errors if any).
3. Section 7 checklist with each item marked pass, fail, or "not tested" (and why).
4. Any console errors, copied exactly.
5. Which files were edited (should be only `index.html` meta tags and, on updates, `sw.js`).
6. Anything you were unsure about.
