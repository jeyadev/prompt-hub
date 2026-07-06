# Prompt 1 — Export Repository → PNG Context Pack (pxpipe)

> **Where to paste:** GitHub Copilot (agent mode) **or** Claude Code, with the
> workspace open at the **repository root**. Both run on Anthropic models.
> This is a drop-in `.prompt.md` — you can also save it under
> `.github/prompts/` (Copilot) or invoke it as a Claude Code command.

---

## Role

You are a build agent. Your job is to serialize this entire repository into a
**PNG context pack** using [`pxpipe`](https://github.com/teamchong/pxpipe)
(`pxpipe-proxy` on npm), so a *separate* agent session can later rebuild the
repo from the images alone.

pxpipe is **lossy on exact strings by design** (its own benchmark: verbatim
hex recall 13/15 on Fable 5, 0/15 on Opus 4.8). You will therefore **not**
naively image everything. You will split the repo into two lanes:

- **SOURCE lane** → rendered to PNG pages (bulk logic, structure, comments).
- **SHARP lane** → copied verbatim as text (byte-exact-critical files the
  vision channel cannot be trusted with).

And you will emit a **manifest + checksums** so rebuild can verify and self-heal.

## Non-negotiable rules

1. Operate only on **git-tracked** files (`git ls-files`) so `.gitignore`,
   `node_modules`, build output, and vendored blobs are excluded automatically.
2. **Never image secrets.** `.env*`, credential files, and anything matching a
   secret shape go to the SHARP lane as-is (or are excluded if the user says so).
   Do not print secret contents to the chat.
3. Every tracked file must end up in **exactly one** lane and be listed in
   `manifest.json` with its `sha256`, byte length, and line count.
4. Do not summarize, reformat, or "clean up" any file. This is serialization.

## Steps

### 0 — Preconditions
- Confirm Node ≥ 18: `node -v`.
- Confirm repo root: `git rev-parse --show-toplevel`.
- Confirm network access to the npm registry.

### 1 — Install pxpipe
```bash
npm i -D pxpipe-proxy
```
(If installs are blocked, `npx pxpipe-proxy` at runtime also works, but a local
devDependency is more reproducible.)

### 2 — Write the export script
Create `scripts/pxexport.mjs` with **exactly** this content:

```js
// scripts/pxexport.mjs — serialize a git repo into a pxpipe PNG context pack.
import { renderTextToImages } from "pxpipe-proxy";
import { execSync } from "node:child_process";
import { readFileSync, writeFileSync, mkdirSync, existsSync, rmSync } from "node:fs";
import { dirname, join, basename } from "node:path";
import { createHash } from "node:crypto";

const OUT   = ".pxexport";
const PAGES = join(OUT, "pages");
const SHARP = join(OUT, "sharp");
const MAX_CHARS_PER_BUNDLE = 80_000; // stay under pxpipe's ~92k chars/page ceiling

// reset output dir
if (existsSync(OUT)) rmSync(OUT, { recursive: true, force: true });
mkdirSync(PAGES, { recursive: true });
mkdirSync(SHARP, { recursive: true });

const sha = (buf) => createHash("sha256").update(buf).digest("hex");
const tracked = execSync("git ls-files", { encoding: "utf8" })
  .split("\n").map((s) => s.trim()).filter(Boolean);

// byte-exact-critical files: the vision channel must not be trusted with these
const SHARP_NAMES = new Set([
  "package-lock.json", "pnpm-lock.yaml", "yarn.lock", "bun.lockb",
  "poetry.lock", "Cargo.lock", "go.sum", "composer.lock", "Gemfile.lock",
]);
const isBinary  = (buf) => buf.includes(0); // NUL byte ⇒ treat as binary
const isSecret  = (p)   => /(^|\/)\.env(\.|$)|(^|\/)\.?(secrets?|credentials?)(\.|\/|$)/i.test(p);
const looksSharp = (p, buf) =>
  SHARP_NAMES.has(basename(p)) || isSecret(p) || isBinary(buf);

const manifest = { tool: "pxpipe", createdAt: new Date().toISOString(),
                   source: [], sharp: [], pages: [] };
const checks = [];
const sources = [];

for (const path of tracked) {
  let buf;
  try { buf = readFileSync(path); } catch { continue; }
  const digest = sha(buf);
  checks.push(`${digest}  ${path}`);            // sha256sum -c compatible

  if (looksSharp(path, buf)) {
    const dst = join(SHARP, path);
    mkdirSync(dirname(dst), { recursive: true });
    writeFileSync(dst, buf);                     // verbatim, never imaged
    manifest.sharp.push({ path, sha256: digest, bytes: buf.length,
      reason: isBinary(buf) ? "binary" : isSecret(path) ? "secret" : "exact-critical" });
  } else {
    const text = buf.toString("utf8");
    const rec  = { path, sha256: digest, bytes: buf.length, lines: text.split("\n").length };
    sources.push({ ...rec, text });
    manifest.source.push(rec);
  }
}

// wrap each source file in unambiguous delimiters that survive rendering
const wrap = (f) =>
  `<<<PXFILE path="${f.path}" sha256="${f.sha256}" bytes="${f.bytes}" lines="${f.lines}">>>\n` +
  f.text +
  `\n<<<PXEND path="${f.path}">>>\n`;

// pack files into bundles under the per-page char ceiling
const bundles = [];
let cur = "", curFiles = [];
for (const f of sources) {
  const block = wrap(f);
  if (cur.length + block.length > MAX_CHARS_PER_BUNDLE && cur.length > 0) {
    bundles.push({ text: cur, files: curFiles }); cur = ""; curFiles = [];
  }
  cur += block; curFiles.push(f.path);
}
if (cur.length) bundles.push({ text: cur, files: curFiles });

// render each bundle → PNG page(s)
let pageNo = 0;
for (let b = 0; b < bundles.length; b++) {
  const { pages } = await renderTextToImages(bundles[b].text);   // pages[i].png: Uint8Array
  for (const pg of pages) {
    const fname = `page-${String(pageNo).padStart(4, "0")}.png`;
    writeFileSync(join(PAGES, fname), Buffer.from(pg.png));
    manifest.pages.push({ file: `pages/${fname}`, bundle: b, files: bundles[b].files });
    pageNo++;
  }
}

writeFileSync(join(OUT, "manifest.json"), JSON.stringify(manifest, null, 2));
writeFileSync(join(OUT, "checksums.sha256"), checks.join("\n") + "\n");

const kb = (n) => (n / 1024).toFixed(0);
console.log(`pxexport ✓  ${sources.length} source files → ${pageNo} PNG pages`);
console.log(`            ${manifest.sharp.length} sharp sidecar files (verbatim)`);
console.log(`            output: ${OUT}/ {pages/, sharp/, manifest.json, checksums.sha256}`);
```

### 3 — Run it
```bash
node scripts/pxexport.mjs
```

### 4 — Self-check the pack (before declaring done)
- Assert every git-tracked file appears **once** across
  `manifest.source[]` + `manifest.sharp[]`:
  ```bash
  node -e "const m=require('./.pxexport/manifest.json');const g=require('child_process').execSync('git ls-files',{encoding:'utf8'}).split('\n').filter(Boolean);const s=new Set([...m.source,...m.sharp].map(x=>x.path));const miss=g.filter(p=>!s.has(p));console.log(miss.length?('MISSING: '+miss.join(', ')):'coverage ✓ '+g.length+' files')"
  ```
- Assert `pages/` is non-empty and `checksums.sha256` has one line per tracked file.
- Spot-open `pages/page-0000.png` and confirm the `<<<PXFILE ...>>>` delimiters
  and code are legible.

### 5 — Report
Print a short table: source files, sharp files, page count, total pack size on
disk, and a rough token estimate (imaged ≈ `pages × 4,761` vision tokens vs
text ≈ `total_source_chars ÷ 4`). State the expected token reduction.

## Hand-off

The `.pxexport/` folder is the deliverable. Ship `pages/`, `sharp/`,
`manifest.json`, and `checksums.sha256` together — the rebuild prompt needs all
four. Do **not** commit secrets in `sharp/`; if any secret files were captured,
replace their contents with placeholders and note it in the report.
