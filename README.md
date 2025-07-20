# tonivega-fuse-html – Browse the Web as a Filesystem (⚡ HTML Link Graph via FUSE)

> **Mount any web page and *cd* through its outgoing links as if the Web were a directory tree.** Each page is a directory. Its body appears as `content.<ext>`. Every discovered resource link (anchor, script, img, iframe, etc.) becomes a subdirectory you can descend into—*recursively and lazily*.

<p align="center"><img src="https://img.shields.io/badge/status-experimental-orange" alt="status" /> <img src="https://img.shields.io/badge/license-freeware-blue" alt="license" /> <img src="https://img.shields.io/badge/go-%3E=1.20-00ADD8" alt="go version" /> <img src="https://img.shields.io/badge/FUSE-bazil.org%2Ffuse-success" alt="fuse" /></p>

## ✨ Why tonivega-fuse-html?

* **Instant Mental Model:** Turn the abstract hyperlink graph into tangible folders you can *ls*, *find*, *grep*, *diff*, *wc*.
* **Zero Browser Needed:** Inspect third‑party script payloads, stylesheets, JSON feeds directly in your terminal/editor.
* **Power of UNIX Pipelines:** `grep`, `ripgrep`, `awk`, `jq`, `fzf`, `du` across live web resources.
* **Lazy, On‑Demand Fetching:** Only pages you traverse are fetched. Recursive exploration emerges naturally.
* **Uniform Handling of Heterogeneous Resources:** HTML, CSS, JS, images, media, JSON—all exposed as simple files.

## 🚀 Quick Start

```bash
# Fetch the tonivega-fuse-html source
# Run it with a mount point and a seed URL:
mkdir /tmp/linkmount
./tonivega-fuse-html /tmp/linkmount https://www.bluengo.com  # sudo may be required on some systems

# Explore!
ls /tmp/linkmount
cat /tmp/linkmount/content.html
```

> **Unmount:** `fusermount -u /tmp/linkmount` (Linux) or `umount /tmp/linkmount` (macOS).

## 🧠 Conceptual Model

```
<URL Directory>
 ├── content.<ext>        # Raw body of the URL response
 ├── child1.example.com   # Another page/resource (directory)
 │    └── content.<ext>
 ├── image.cdn__path_img  # Image resource dir (contains content.jpg)
 │    └── content.jpg
 └── script.cdn__libs     # Script resource dir (contains content.js)
      └── content.js
```

**Rule:** If the MIME type is `text/html`, tonivega-fuse-html parses outbound links to create further subdirectories. Non‑HTML resources become leaf directories with only their `content.<ext>`.

## 🔍 What Gets Parsed

Tags & attributes scanned:

| Tag      | Attribute | Notes                                     |
| -------- | --------- | ----------------------------------------- |
| `a`      | `href`    | Relative & absolute resolved              |
| `link`   | `href`    | Stylesheets, preloads, etc.               |
| `script` | `src`     | External scripts                          |
| `img`    | `src`     | Images                                    |
| `iframe` | `src`     | Embedded frames (⚠ network depth)         |
| `source` | `src`     | `<picture>`, `<video>`, `<audio>` sources |
| `video`  | `src`     | Direct media                              |
| `audio`  | `src`     | Direct audio                              |

Only `http` / `https` links are retained. Duplicates within the same page are merged.

## 🧩 Name Sanitization

URLs are transformed into filesystem-friendly names:

* `scheme://` stripped
* Host + path combined (`/` → `__`)
* Query → `__q_<sanitized>`; Fragment → `__f_<sanitized>`
* Non `[A-Za-z0-9._-]` replaced with `_`
* Length truncated to 120 chars
* Collisions resolved by `_2`, `_3`, ... suffixes

## 🗃 File Naming & Extensions

Each directory houses exactly one body file: `content.<ext>` where `<ext>` is guessed by:

1. Known MIME → extension map
2. Path suffix (URL extension)
3. `mime.ExtensionsByType`
4. Fallback heuristics (`html`, `txt`, `dat`)

## 🕸 Traversal Strategy

* *Lazy*: A page fetch occurs on first `lookup`/`ls`.
* *Depth Emergent*: Depth increases only as you visit new directories.
* *In-Memory Cache*: Node bodies & link sets persist until unmount.

## 🛡 Read-Only Guarantee

No mutation operations are implemented—safe for use in investigative workflows.

## ⚙ Architecture Overview

```
+-------------+        +------------------+        +-------------------+
|   FUSE VFS  | <----> |   tonivega-fuse-html (FS)    | <----> |  Node Registry     |
+-------------+        +------------------+        +-------------------+
       |                         |                           |
       | readdir/lookup/read     | ensures fetch             | urlStr -> *Node
       v                         v                           v
   Kernel FUSE            Lazy Fetch & Parse          In-Memory Cache
                              |      \
                              |       +--> HTML Tokenizer
                              |            (extract outbound links)
                              v
                      Body + MIME + Children
```

## 🧪 Example Power Moves

| Goal                                              | Command                                                     | Result                    |                |                  |
| ------------------------------------------------- | ----------------------------------------------------------- | ------------------------- | -------------- | ---------------- |
| Find every external JS URL containing `analytics` | `grep -R "analytics" /tmp/linkmount --include='content.js'` | Surfaces tracking scripts |                |                  |
| Count total bytes fetched so far                  | \`du -ch /tmp/linkmount                                     | tail -n1\`                | Quick size sum |                  |
| List all unique domains visited                   | \`find /tmp/linkmount -name 'content.\*' -printf '%h\n'     | cut -d/ -f4               | sort -u\`      | Domain inventory |
| Extract all JSON bodies                           | `find /tmp/linkmount -name 'content.json' -exec cat {} +`   | Aggregate API payloads    |                |                  |
| Grep inline HTML for TODOs                        | `grep -R "TODO" /tmp/linkmount --include='content.html'`    | Developer hints           |                |                  |

## 🐛 Troubleshooting

| Symptom                      | Cause                                      | Fix                                                                  |
| ---------------------------- | ------------------------------------------ | -------------------------------------------------------------------- |
| `permission denied` mounting | Not enough privileges                      | Use `sudo` or ensure user in `fuse` group                            |
| Immediate unmount            | Missing FUSE kernel module                 | `modprobe fuse` (Linux)                                              |
| Missing links                | Page loaded via JS (client-side rendering) | Tool does not execute JS; supply server-rendered pages or pre-render |
| Very deep / huge tree        | Highly connected site                      | Limit traversal depth manually, avoid `find -R` on massive domains   |
| Non-UTF8 gibberish           | Binary response                            | That's expected; file still `content.<ext>`                          |

## 🧱 Limitations (Current)

* No persistence or on-disk caching.
* No HEAD/ETag revalidation; each node fetched at most once per mount session.
* No JavaScript execution (static HTML only).
* Potential name collisions on extremely similar URLs (mitigated but not cryptographic).
* Not sandboxed; fetched resources are held in memory—avoid mounting untrusted giant blobs.

## 🔐 Security Considerations

* Avoid mounting behind authenticated sessions (no cookie/jar logic currently).
* Treat fetched third-party code as untrusted; you're *reading* only, but still mindful of supply chain.

## 🗺 Roadmap Ideas

* [ ] Ask support@bluengo.com

## 🧩 Integrations / Hacks

| Idea                     | Description                                                                            |
| ------------------------ | -------------------------------------------------------------------------------------- |
| *Greppable Threat Intel* | Mount suspicious landing page, grep for known IoCs in scripts.                         |
| *Asset Inventory*        | List all script & stylesheet domains a site depends on.                                |
| *Content Diffing*        | Mount same URL via different network vantage points (VPN/proxy) → diff `content.html`. |
| *Performance Audits*     | Enumerate third-party resource explosion by counting directories.                      |
| *Automated Harvest*      | `find` + `xargs` pipeline to copy subset of resource bodies for offline analysis.      |


## 🧾 License

Freeware – do anything, just retain the license notice.

## 🙌 Contributing

PRs & discussions welcome: refine naming, add flags, optimize memory, extend parsers.

## 💡 Inspiration

FUSE’s power + the UNIX philosophy: *"Everything is a file"* – now, *so is the hyperlink graph.*

> **Go mount the Web.** 🌐 `cd` is your browser now.
