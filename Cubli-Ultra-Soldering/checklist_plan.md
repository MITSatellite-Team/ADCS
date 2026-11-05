# Implementation spec: persist soldering progress to committable files

**Audience: the AI agent implementing this.** Written to be executed without further design decisions. All design questions have been settled with the repo owner; §1 records the facts, §2 the decisions, §3–§6 the work, §7 verification. Deviating from §2 requires asking the owner first.

Status: **not yet implemented.** Execute §3 onward on instruction.

---

## 1. Established facts

Verified by direct inspection of the files. Do not re-derive; do re-confirm anything a step depends on before editing.

### 1.1 Paths

| Thing | Path |
|---|---|
| Repo root | `/Users/beaveracosta/School/MIT Sat/ADCS` |
| Work dir | `/Users/beaveracosta/School/MIT Sat/ADCS/Cubli-Ultra-Soldering` |
| Remote | `https://github.com/MITSatellite-Team/ADCS.git`, branch `main` |
| Board helpers | `{main,encoder,imu,programmer}-soldering.html` |
| Entry page | `START_HERE.html` |
| Schematics | `reference/*.pdf` — do not touch |

Repo root has **no `.gitignore`**. Working tree was clean at inspection.

### 1.2 Structure of the four board files

Each is 132 lines. Line numbers are stable and identical across all four:

| Line | Content |
|---|---|
| 7 | `<style>` block, one long line. Contains `tbody tr.checked .ref-button{text-decoration:line-through;color:#61766c}` |
| 26 | `<tbody id="parts"></tbody>` — ships **empty** |
| 28 | `.checks-tools` div: `#export` and `#reset` buttons, then `<p class="help-line" id="storage-note">` |
| 32 | `<script id="board-data" type="application/json">` — the whole BOM as one line (498 KB in `programmer`, 2.5 MB in `main`) |
| 33 | `<script>` — start of app code |
| 35 | `const DATA=JSON.parse(document.getElementById('board-data').textContent);` |
| 40 | `let selected=...,side=selected.side,view=[...DATA.viewBox],checked=new Set();` |
| 41 | `const key='cubli-ultra-assembly-'+DATA.version+'-'+DATA.board;` |
| 42 | localStorage load — **replaced in §4.2** |
| 48 | `function save(){...}` — **replaced in §4.4** |
| 103–110 | `function renderList(){...}`, progress text at 110 |
| 123 | `$('reset').onclick=...` |
| 124 | `$('export').onclick=...` — CSV export, **leave alone** |
| 129 | `window.assemblyHelper={...}` |
| 130 | `draw();showSelection();renderList();` + ResizeObserver |

**Critical property: lines 33–132 are byte-identical in all four files** (same MD5). Every `old_string` in §4 therefore matches exactly once per file, and the same edit applies to all four. The only board-specific edit is the loader filename in §4.1.

Excluding line 32, the files differ only in title and board name.

### 1.3 There is no checked-state in the HTML — correcting the original premise

The owner's initial description said `programmer-soldering.html` contains table rows with a `checked` class recording soldered parts. **It does not.** Verified:

- The parts table ships empty (line 26); every row is built at runtime by `renderList()` (line 103).
- `checked` is CSS (line 7) plus a class toggled live per render: line 105, `tr.classList.toggle('checked',checked.has(c.ref))`.
- The string `checked` occurs **28 times in all four files identically**, including `imu-soldering.html`, which has no soldered parts. It is framework code, not data.

**Actual location of existing progress:** `localStorage`, written by `save()` (line 48) under

```
'cubli-ultra-assembly-' + DATA.version + '-' + DATA.board
→ cubli-ultra-assembly-2026-10-30-v1-programmer
```

read back at line 42. Single browser profile, single machine, invisible to git, and silently discarded on any `version` bump because the key embeds the version. That is the problem this spec fixes.

**Implication:** there is nothing to scrape from the HTML. The pre-existing ticks were exported from the browser by the owner before any patching; the data is in §2.4. That migration is complete — do not re-run it.

### 1.4 Board data

All four carry `"version":"2026-10-30-v1"`. Use `DATA.version` at runtime, never a literal.

| Board | fit | other | Initial selection |
|---|---|---|---|
| `main` | 229 | `wire`:1, `dnp`:2 | `U15` |
| `encoder` | 3 | `omit`:3 | `U3` |
| `imu` | 5 | `omit`:1 | `U1` |
| `programmer` | 31 | `omit`:2 | `U4` |

Only `status:"fit"` positions are checkable. 268 fit positions total.

### 1.5 Browser constraints that drove the design

From a double-clicked `file://` page:

| Mechanism | Available? |
|---|---|
| `<script src="relative.js">` | ✅ all browsers |
| `fetch('relative.json')` | ❌ CORS — origin is `null` |
| `XMLHttpRequest`, relative path | ❌ Chrome, Firefox |
| File System Access (`showSaveFilePicker`) | ❌ not on `file://`; Chromium-only on http |
| `<input type="file">` manual pick | ✅ all browsers |
| Blob + `a.download` | ✅ all browsers → `~/Downloads` |

**A strict `.json` file cannot be auto-loaded by a double-clicked page.** Browser security rule, not a code problem. This forced the §2.1 decision.

---

## 2. Settled decisions — implement exactly

### 2.1 Transport: `.js` wrapper, loaded by `<script src>`

Decided by the owner over two alternatives (strict `.json` via `fetch` + local server; strict `.json` picked by hand each session). Both were rejected for requiring either a server step or a per-session manual file-pick at a workbench. Owner's stated requirement: state must live outside the browser and outside the HTML; the file extension is immaterial.

Additionally, when the page happens to be served over `http://`, use File System Access so Save writes the file directly instead of going through `~/Downloads`. One code path, origin-detected. **No server is ever required.**

### 2.2 File format — exact bytes

One file per board at `progress/<board>.js`. `progress/programmer.js` after seeding:

```js
window.SOLDER_PROGRESS = window.SOLDER_PROGRESS || {};
window.SOLDER_PROGRESS["programmer"] = {
  "schema": 1,
  "board": "programmer",
  "boardVersion": "2026-10-30-v1",
  "checked": {
    "C7": true,
    "C8": true,
    "C9": true,
    "R12": true,
    "R13": true,
    "R14": true,
    "R15": true,
    "R16": true,
    "R17": true,
    "R18": true,
  },
};
```

Mandatory properties:

1. **`"REF": true`, one per line.** Lean form, per owner instruction — no `by`/`at` attribution. Git already records the commit author.
2. **Trailing comma after every entry, including the last.** See §2.3 — this is load-bearing, not sloppiness.
3. **Object, not array.** Duplicate keys from a botched merge collapse to one value; a duplicated array element would not.
4. **Canonical order: alphabetic letter-prefix, then *numeric* digits.** `R2, R12, R16, R17` — not the `R12, R16, R2` a plain string sort gives. Required for bench readability and byte-stability.
5. **No timestamps anywhere in the file.** A `savedAt` field would make every save produce a diff and destroy the no-op-save property.
6. **`boardVersion` mirrors `DATA.version`.** Used for the §4.3 warning, never to discard data.
7. Trailing newline, `\n` endings, 2-space indent.

Rationale for one-file-per-board and one-line-per-ref, both chosen for git behavior in a shared repo: two people on different boards never touch the same file; two people on the *same* board edit different lines, which git auto-merges. A single-line array like `["C7","C8"]` would conflict on every concurrent tick. Matters most for `main` at 229 positions.

### 2.3 Why trailing commas on every line — revised decision, flag to owner

The earlier draft kept the body strictly JSON-parseable (strip the wrapper lines, feed to `jq`), which meant **no** comma after the final entry. Combined with the lean format, that creates a real defect: appending a new alphabetically-last ref requires editing the *previous* last line to add a comma. Two people each adding a last-sorting ref both modify that same line → guaranteed merge conflict, in exactly the workflow this spec exists to support.

The file is executed as JavaScript, and **JS object literals permit trailing commas.** Emitting a comma after every entry means any insertion or removal touches exactly one line, so concurrent ticks always auto-merge.

Cost: the body is no longer strictly valid JSON, so `jq` needs the final comma stripped first:
```sh
sed -n '3,$p' progress/programmer.js | sed '$d' | sed 's/,\( *[}]\)/\1/' | jq .
```

This trades a speculative convenience (`jq` on a 10-line file) for the primary design goal (clean merges). **Implement trailing commas.** Mention the change when reporting completion so the owner can override; reverting means re-running the serializer, not rewriting data.

### 2.4 Seed data — migration already captured

`programmer` is the **only** board with pre-existing checkmarks; owner confirmed the other three are empty. Exported via `assemblyHelper.checked` (the getter at line 129) before any patching:

```json
["C7","C8","C9","R12","R13","R14","R15","R17","R18","R16"]
```

10 refs. Note `R16` arrives last in the export — insertion order is irrelevant, the serializer imposes §2.2 rule 4. Expect `R15, R16, R17, R18` in the file.

Create `progress/{main,encoder,imu}.js` with an empty `"checked": {}` — present rather than absent, so the first person to tick a part edits a file that already exists in the documented shape.

### 2.5 Not written: `sync-progress.sh`

Considered and declined. A script guessing at `~/Downloads` filenames risks a confusing mis-copy; the explicit `mv` line in the Save button's helper text is clearer. Do not add one.

---

## 3. Preconditions

Before editing anything:

1. **Shell access is required** from §7 onward (git merge test, byte-comparison). The session that produced this spec had Bash blocked by a session-level safety check partway through; if that is still the case, file edits can proceed but §7 cannot. Do not report completion without §7.
2. `cd` to the work dir and confirm `git status` is clean. If not, stop and report — these edits touch four large generated files and a dirty tree makes them hard to back out.
3. Confirm the §1.2 line numbers still hold (`sed -n '42p'` and `sed -n '48p'` should match the strings in §4.2 and §4.4). If they have shifted, re-locate by string rather than by line and note the discrepancy.

---

## 4. Task: patch the four board HTML files

Apply **identical** edits to all four. The `old_string`s below each match exactly once per file (§1.2). Prefer the editing tool over a script; if scripting, use literal string replacement, not regex.

**Do not reformat, re-minify, or reflow these files.** Every edit is a surgical insert or single-line replacement. Do not touch line 32.

### 4.1 Add the loader tag

Insert immediately before line 33's `<script>`, i.e. between the data block and the app code. The filename is the **only** board-specific edit:

```html
<script src="progress/main.js"></script>
<script src="progress/encoder.js"></script>
<script src="progress/imu.js"></script>
<script src="progress/programmer.js"></script>
```

Must precede the app script so `window.SOLDER_PROGRESS` exists at init. A missing file 404s silently and leaves the global undefined, which §4.2 handles as "no file".

### 4.2 Replace the state loader

Replace the whole of line 42:

```js
try{const saved=JSON.parse(localStorage.getItem(key)||'[]');if(Array.isArray(saved)) checked=new Set(saved.filter(r=>byRef.get(r)?.status==='fit'));}catch{}
```

with:

```js
const FILE_STATE=(window.SOLDER_PROGRESS||{})[DATA.board]||null;
const isFit=r=>byRef.get(r)?.status==='fit';
let dirty=false,fileRefs=[],droppedRefs=[],staleRefs=[],versionMismatch=false,lastSaveMsg='';
if(FILE_STATE&&FILE_STATE.checked&&typeof FILE_STATE.checked==='object'){
 const all=Object.keys(FILE_STATE.checked).filter(r=>FILE_STATE.checked[r]===true);
 fileRefs=all.filter(isFit);droppedRefs=all.filter(r=>!isFit(r));
 versionMismatch=FILE_STATE.boardVersion!==DATA.version;
}
const fileStamp=DATA.version+':'+fileRefs.length+':'+fileRefs.slice().sort().join(',');
let buf=null;
try{const raw=JSON.parse(localStorage.getItem(key)||'null');
 if(raw&&Array.isArray(raw.refs))buf=raw;else if(Array.isArray(raw))buf={refs:raw,base:null};
}catch{}
if(buf&&(buf.base===fileStamp||!FILE_STATE)){checked=new Set(buf.refs.filter(isFit));dirty=true;}
else if(buf){checked=new Set(fileRefs);staleRefs=buf.refs.filter(r=>isFit(r)&&!checked.has(r));}
else checked=new Set(fileRefs);
```

Behavioral requirements this encodes:

1. Committed file is the baseline.
2. `localStorage` is retained as an **unsaved-work buffer** so a crash mid-session doesn't lose an hour of ticking, and so the page works before anything has been saved.
3. The existing validity filter (`status==='fit'`) is preserved, but drops are now **reported** (§4.6) instead of silent — on a shared file, a dropped ref means someone else's recorded work was just invalidated.
4. `boardVersion` mismatch **loads the data anyway and warns.** Never reset on a version bump; that is the current localStorage bug.
5. Legacy bare-array buffers (what exists on the owner's machine right now) are read via the `Array.isArray(raw)` branch with `base:null`.

### 4.3 Reconciliation rule

Encoded above; stated here because it is the subtlest part. The buffer stores the refs **plus a stamp of the file it was based on**. A blind union would be wrong — unchecking a part would be silently undone by a stale buffer.

| Condition | Winner | UI |
|---|---|---|
| Buffer stamp **matches** file | Buffer — unsaved newer work | mark unsaved |
| Buffer exists, **no** file | Buffer | mark unsaved |
| Buffer stamp **differs** (someone pulled) | **File** | list refs only in the stale buffer, by ref |
| No buffer | File | — |

Never silently discards work; never silently resurrects it. Any ref that disappears is named on screen.

### 4.4 Replace `save()` and add the serializer and file writer

Replace the whole of line 48:

```js
function save(){try{localStorage.setItem(key,JSON.stringify([...checked]));}catch{$('storage-note').textContent='This browser could not save checkmarks. Export the checklist to keep a record.';$('storage-note').classList.add('storage-error');}}
```

with:

```js
function save(){dirty=true;try{localStorage.setItem(key,JSON.stringify({refs:[...checked],base:fileStamp}));}catch{$('storage-note').textContent='This browser could not buffer checkmarks. Use Save progress to file now, or you may lose this session.';$('storage-note').classList.add('storage-error');}}
function cmpRef(a,b){const m=/^([A-Za-z]*)(\d*)/,x=m.exec(a),y=m.exec(b),p=x[1].toUpperCase(),q=y[1].toUpperCase();
 if(p!==q)return p<q?-1:1;const n=(x[2]?parseInt(x[2],10):-1)-(y[2]?parseInt(y[2],10):-1);return n||(a<b?-1:a>b?1:0);}
function serialize(){
 const refs=[...checked].sort(cmpRef);
 return 'window.SOLDER_PROGRESS = window.SOLDER_PROGRESS || {};\n'
 +'window.SOLDER_PROGRESS['+JSON.stringify(DATA.board)+'] = {\n'
 +'  "schema": 1,\n'
 +'  "board": '+JSON.stringify(DATA.board)+',\n'
 +'  "boardVersion": '+JSON.stringify(DATA.version)+',\n'
 +'  "checked": {\n'
 +refs.map(r=>'    '+JSON.stringify(r)+': true,\n').join('')
 +'  },\n};\n';
}
async function writeFile(){
 const text=serialize();
 if(location.protocol!=='file:'&&window.showSaveFilePicker){
  try{const h=await window.showSaveFilePicker({suggestedName:DATA.board+'.js',startIn:'documents',
    types:[{description:'Progress file',accept:{'text/javascript':['.js']}}]});
   const w=await h.createWritable();await w.write(text);await w.close();
   dirty=false;try{localStorage.removeItem(key);}catch{}
   lastSaveMsg='Saved to file. Commit progress/'+DATA.board+'.js when you are done.';
   renderList();showStorageNote();return;}
  catch(e){if(e.name==='AbortError')return;}
 }
 const blob=new Blob([text],{type:'text/javascript'}),url=URL.createObjectURL(blob),a=document.createElement('a');
 a.href=url;a.download=DATA.board+'.js';a.click();setTimeout(()=>URL.revokeObjectURL(url),1000);
 dirty=false;
 lastSaveMsg='Downloaded. Now run:  mv ~/Downloads/'+DATA.board+'.js progress/'+DATA.board+'.js   then commit it.';
 renderList();showStorageNote();
}
```

Requirements:

- `save()` keeps buffering on **every** tick, as today, and marks dirty. Writing the file is explicit — every available write path needs a user gesture.
- **The localStorage buffer is cleared only on a true in-place write.** On the download path the buffer is deliberately retained: the file is still sitting in `~/Downloads` and clearing it would lose the work if the page were reloaded before the `mv`. Do not "simplify" this.
- Serializer output must be byte-stable: sorted, no timestamps. Saving twice with no changes must produce an identical file.

### 4.5 Add the Save button

In line 28's `.checks-tools`, add before `#export`:

```html
<button id="save-file">Save progress to file</button>
```

Wire it near line 123:

```js
$('save-file').onclick=writeFile;
```

Place it first so it reads as the primary action; the existing CSV `#export` is a secondary record-keeping tool.

### 4.6 Add the notice renderer

Define alongside §4.4 and call it once during init (append to line 130's init sequence):

```js
function showStorageNote(){
 const n=$('storage-note'),parts=[];
 if(versionMismatch)parts.push('WARNING: progress file was written for board data '+(FILE_STATE&&FILE_STATE.boardVersion)+' but this page is '+DATA.version+'. Progress was loaded anyway — verify it against the board.');
 if(droppedRefs.length)parts.push('Ignored '+droppedRefs.length+' entry/entries no longer fit-positions on this board: '+droppedRefs.join(', ')+'.');
 if(staleRefs.length)parts.push('Your browser had unsaved marks not in the committed file: '+staleRefs.join(', ')+'. The file won. Re-tick and save if they were real.');
 if(lastSaveMsg)parts.push(lastSaveMsg);
 if(dirty&&!lastSaveMsg)parts.push('Unsaved changes — click Save progress to file, then commit.');
 n.textContent=parts.length?parts.join(' '):'Checkmarks load from progress/'+DATA.board+'.js and are buffered in this browser until you save. They record placement only — not solder-joint quality.';
 n.classList.toggle('storage-error',versionMismatch||staleRefs.length>0);
}
```

Replaces the existing static text, which claims checkmarks are browser-local and becomes false after this change.

### 4.7 Dirty-state guard

Append near line 130:

```js
window.addEventListener('beforeunload',e=>{if(dirty){e.preventDefault();e.returnValue='';}});
```

Without this, the most likely failure is someone ticking 40 parts, closing the tab, and assuming it saved because the checkboxes persisted across reloads all along (via localStorage).

Also, in `renderList()` at line 110, append an unsaved marker to the progress text:

```js
$('progress').textContent=`${checked.size} / ${DATA.counts.fit} checked`+(dirty?' · unsaved':'');
```

### 4.8 Make Clear checkmarks dirty

Line 123:

```js
$('reset').onclick=()=>{if(confirm('Clear checkmarks for this board?')){checked.clear();save();renderList();draw();showStorageNote();}};
```

`save()` now sets dirty, so a clear that is never written to the file is visibly unsaved rather than appearing to work and then being undone by the next pull.

### 4.9 Extend the debug handle

Line 129 — add to the existing object, keep every current member:

```js
window.assemblyHelper={data:DATA,get side(){return side},get selected(){return selected.ref},get checked(){return [...checked]},get dirty(){return dirty},get file(){return FILE_STATE},serialize,select,xy};
```

### 4.10 Out of scope — do not modify

The SVG board viewer, `draw()`, `select()`, `wireTarget()`, all pan/zoom/wheel handlers, the CSV export at line 124, the BOM data at line 32, the rework notes, and `reference/`.

---

## 5. Task: create the progress files

Create `progress/` and the four files per §2.2.

**Seed `progress/programmer.js`** with the 10 refs from §2.4, in canonical order, exactly as shown in §2.2.

**Two checks before writing it** — report results, do not skip:

1. **Every one of the 10 refs exists in `programmer`'s line-32 data block with `status:"fit"`.** A ref failing this would be silently dropped by §4.2's filter. All 10 look plausible against the 31 fit positions, but verify against the data rather than assuming. If any fails, stop and report.
2. **Count check:** 10 of 31, which the UI must render as `10 / 31 checked`. The owner should confirm this matches what their browser shows; if it does not, the original export was incomplete.

**`progress/{main,encoder,imu}.js`** get `"checked": {}` — same wrapper, empty object.

---

## 6. Task: repo-level files

### 6.1 `progress/README.md`

Explicitly requested by the owner. Cover:

- What these files are and that they are the authoritative record.
- Why `.js` and not `.json` (§1.5 — `fetch` is CORS-blocked at `file://`), and the §2.3 `sed`+`jq` line for anyone wanting strict JSON.
- The merge property: one line per ref, trailing commas, so concurrent ticks auto-merge. If git *does* report a conflict, keeping both sides is almost always correct.
- Workflow: `git pull` → open the HTML → tick → **Save progress to file** → `mv` if downloaded → commit → push.
- That the optional `python3 -m http.server` path enables write-in-place.
- **The caveat, verbatim in substance from `START_HERE.html`:** checkmarks record that a part was placed, not that the joint is good. Making this data shared and durable risks making it look more authoritative than it is. Repeat it here.

### 6.2 Update `START_HERE.html`

Two statements become false and must change:

- Line 28 area / `#storage-note` equivalent in the prose: *"Checkmarks are saved in this browser for this board"* → now committed to `progress/`.
- *"export the checklist when handing work over"* → committing is now the handoff.

Add the pull/tick/save/commit workflow, since this is the documented entry point. Match the existing terse house style; do not restructure the page.

### 6.3 `.gitignore`

Repo root has none. Create one with at least `.DS_Store` — shared macOS-authored repo.

---

## 7. Verification — run before reporting completion

Needs shell access (§3.1). Report every result, including failures.

1. **Round-trip.** Tick 3 parts, Save, clear `localStorage` for that key, reload. Exactly those 3 come back, from the file.
2. **Byte-stability.** Save twice with no changes between. `git diff` must be empty.
3. **Merge test — the one that validates the whole format choice.** Branch; tick `C7` on one branch and `C8` on another; merge. Git must resolve with **no conflict** and both must survive. Then repeat with two refs that both sort *last* (e.g. `R18` and `R19` on a board that has them) — this is the case §2.3's trailing commas exist for. If either fails, the format is wrong: stop, report, and revisit §2.2 before proceeding.
4. **Stale-buffer path.** Hand-edit a progress file to simulate a pull while a buffer exists. File must win; orphaned refs must be named on screen.
5. **Version-bump path.** Change `boardVersion` in a progress file. Progress must still load, with the warning shown. Nothing discarded.
6. **All four boards load,** including `main` at 229 positions. Check the browser console is clean on each.
7. **Both origins.** Double-clicked `file://` *and* `python3 -m http.server`. Confirm the File System Access path engages only on http.
8. **Validity filter.** Put a bogus ref and an `omit`-status ref in a progress file. Both dropped, both reported via §4.6.
9. **Patch uniformity.** Confirm the four patched files still differ only in board name, title, line 32, and the §4.1 filename:
   ```sh
   for f in main encoder imu programmer; do
     sed -n '33,$p' "$f-soldering.html" | md5
   done
   ```
   All four must match. If they diverge, an edit was applied inconsistently.
10. **Owner's pre-existing buffer — requires the owner's browser.** Their `localStorage` still holds the 10 refs under `cubli-ultra-assembly-2026-10-30-v1-programmer`. With the file seeded, §4.3's stamp-mismatch branch applies: file wins, `staleRefs` empty (file ⊇ buffer), result 10 refs. Verify no double-count, no drop, nothing resurrected. **Hand this to the owner** — it is the only machine with a real legacy buffer and the easiest case to get subtly wrong.

Steps 1–9 are the agent's. Step 10 is the owner's.

---

## 8. Notes and non-concerns

- **Regeneration is not a risk.** The owner confirmed the four HTML files were generated once from consolidated checklist files and will not be regenerated, so patching them directly is safe and there is no upstream generator to coordinate with. (State still belongs in `progress/` for the §2.2 merge and diff reasons, independent of this.)
- **`main-soldering.html` is 2.5 MB** — base64 SVG in its data block. Slow to diff; a second reason progress lives outside it.
- **The owner's existing ticks are already safe**: §2.4 is committed in this document, so they no longer depend on a browser profile surviving. That was the only unrecoverable step and it is behind us.
- **Do not commit.** The owner authors their own soldering record. Stage nothing, push nothing, unless asked.
- **Report when done:** §7 results, the §5 seed checks, and the §2.3 trailing-comma change so the owner can override it.

---

## 9. Execution order

1. §3 preconditions.
2. §5 — seed checks, then create `progress/` and the four files.
3. §4.1–4.9 — patch all four HTML files.
4. §6 — `progress/README.md`, `START_HERE.html`, `.gitignore`.
5. §7 steps 1–9; report.
6. Hand §7 step 10 to the owner. Leave the commit to them.
