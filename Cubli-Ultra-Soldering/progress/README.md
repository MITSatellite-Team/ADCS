# Soldering progress

The files in this directory (`progress/<board>.js`) are the **authoritative,
committed record** of which parts have been soldered onto each board. They
are loaded by the corresponding `<board>-soldering.html` checklist page and
are tracked in git like any other source file.

## Why `.js` and not `.json`

These pages are opened directly from disk (`file://...`), not served by a
web server. A double-clicked page cannot `fetch()` a sibling file — the
browser blocks it as a cross-origin request when the origin is `null`. A
plain `<script src="progress/programmer.js">` tag has no such restriction,
so the progress data is wrapped as an assignment to `window.SOLDER_PROGRESS`
and loaded like any other script. The `.js` extension is a browser-security
workaround, not a hint about the file's structure — treat the body as data.

If you want strict JSON (e.g. to pipe through `jq`), strip the wrapper lines
and the trailing comma on the last entry first:

```sh
sed -n '3,$p' progress/programmer.js | sed '$d' | sed 's/,\( *[}]\)/\1/' | jq .
```

  ## Workflow
  
  1. `git pull`
  2. Open the board's HTML checklist page
  3. Tick parts as you solder them
  4. Click **Save progress to file**
     - If the page is served over `http://` (e.g. `python3 -m http.server`),
       this writes `progress/<board>.js` in place.
     - If the page is opened as `file://` (double-clicked), the browser
       downloads the file instead — move it into place:
       `mv ~/Downloads/<board>.js progress/<board>.js`
  5. Commit and push
  
## Format

```js
window.SOLDER_PROGRESS = window.SOLDER_PROGRESS || {};
window.SOLDER_PROGRESS["programmer"] = {
  "schema": 1,
  "board": "programmer",
  "boardVersion": "2026-10-30-v1",
  "checked": {
    "C7": true,
    "C8": true,
  },
};
```

One `"REF": true` entry per line, in canonical order (letter prefix, then
numeric digits — `R2, R12, R16`, not a plain string sort). Every entry,
**including the last**, ends with a trailing comma.


### Why trailing commas on every line

This is the detail that makes the format work for a shared repo most of the
time. One ref per line means two people ticking different parts touch
different lines, so git merges them automatically — as long as those lines
land at different points in the file. If the last entry had no trailing
comma (stricter JSON), adding a new alphabetically-last ref would require
editing the *previous* last line to add a comma, which guarantees a
conflict every time two people's newest ref both sort last. JS object
literals allow a trailing comma after the final property, so a normal
insertion only touches the one line being added, not the line before it.

### When you'll still see a merge conflict

Git conflicts when both sides insert immediately after the *same* existing
line — which still happens here if two people's newest checked ref both
happen to sort alphabetically last (e.g. you check `R19` and a teammate
checks `USB2`, and both were appended right after the previous last entry,
`R18`). Trailing commas don't prevent this specific case: git's merge is
line-based, and both insertions are "add a line right after `R18`," which
is exactly the kind of overlapping edit git conflicts on rather than
guesses at ordering. Inserts at different points in the file (anything not
tied for last) still merge cleanly with no manual step.

When this happens, `git merge`/`git pull` will show something like:

```
    "R17": true,
    "R18": true,
<<<<<<< HEAD
    "R19": true,
=======
    "USB2": true,
>>>>>>> their-branch
  },
};
```

**To resolve: keep both lines**, in canonical order (§Format), and delete
the conflict markers. For the example above:

```js
    "R17": true,
    "R18": true,
    "R19": true,
    "USB2": true,
  },
};
```

Never pick just one side here — each side represents a real part someone
soldered. Then `git add` the file and continue the merge/rebase as usual.
This is the only manual step this format requires; it only comes up for the
alphabetically-last ref, and the fix is always "keep both, reorder if
needed."


## Caveat

Checkmarks record that a part was **placed**, not that the solder joint is
good or that the circuit works. Making this record shared and durable
should not make it look more authoritative than it is — verify joints and
function the same way you would regardless of what this file says.
