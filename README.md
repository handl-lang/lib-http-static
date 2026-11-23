# lib-http-static

`http-static` — static file serving for Handl HTTP servers, in pure
handl. The third library in the P-007 HTTP stack (after
`lib-http-route` routing and `lib-http-middleware` onion
middleware): it answers `GET`/`HEAD` requests for files under a
configured root directory and returns the `lib-http-rfc` response
record `{ status?, headers?, body? | body_bytes? }`, so it drops
straight into `serve`/`serve-epoll` handlers and into middleware
chains.

```handl
from "http-static" import { static_files }
from "serve-epoll" import { serve }

-- serve ./public on GET/HEAD; everything else -> 405
serve(static_files("./public"), { port: 8080 })

-- opt-in directory listing + dotfiles
serve(static_files("./public", { listing: true, dotfiles: true }),
       { port: 8080 })
```

## Surface

- `static_files(root, opts?)` → `(req) ~> resp` — the handler.
- `mime_of(path)` → Content-Type string (extension map: html, css,
  js, mjs, json, png, jpg, jpeg, gif, svg, wasm, ico, txt, md, map,
  webp, avif, woff, woff2, ttf, otf, pdf, csv, xml, and a few more).
- `http_date(ms)` → IMF-fixdate string (`Last-Modified`).
- `safe_rel(path)` → `{ segs } | { bad }` — the testable path
  normalization core.
- `default_fs` → the built-in fs port record (exported so callers can
  wrap or extend it).

`opts` (all optional):

- `fs` — filesystem port record
  `{ stat, lstat, read_file, list_dir, path_exists, is_dir, is_file }`
  (each returning the shapes `posix-c/fs` uses). Default: a built-in
  libc backend driven through `ffi.call` (same mechanism as
  `posix-c`'s `fs.hnl`, trimmed to the needed surface). Inject your
  own for non-libc hosts or tests.
- `index` — directory index file name, default `"index.html"`.
- `listing` — `Bool`, default `false`; render a simple HTML index for
  directories with no index file.
- `dotfiles` — `Bool`, default `false`; serve segments starting with
  `.` (default hides them → 404).
- `headers` — record merged into every file response's headers
  (e.g. `{ "cache-control": "public, max-age=3600" }`).

## Semantics

- **Traversal safety** — the request path is percent-decoded, then
  split into segments; empty and `.` segments are dropped, `..`
  segments are rejected outright, and NUL characters or malformed
  `%` escapes reject the request. The surviving segments are joined
  under `root`; absolute paths and encoded slashes (`%2f`) can never
  escape because decoding happens before splitting and the join is
  always `root + "/" + seg/seg/...`.
- **Symlinks** — `lstat` is used for the final component; a symlink
  anywhere that would be followed is rejected with 403 (a symlink
  whose target stays inside root is still rejected — keep the rule
  simple and predictable).
- **Status codes** — `200` file, `206` partial (single `Range`
  header: `bytes=a-b`, `bytes=a-`, `bytes=-n`; unsatisfiable → `416`
  with `Content-Range: bytes */size`), `304` on `If-None-Match`
  matching the weak ETag (`*` and lists handled), `301` redirect for
  directories missing a trailing slash (query string preserved),
  `403` for unsafe paths / symlink escapes, `404` for missing files
  and hidden dotfiles, `405` for non-GET/HEAD methods with
  `Allow: GET, HEAD`.
- **Headers** — `Content-Type`, `Content-Length` (also on HEAD),
  `Last-Modified`, `ETag` (`W/"<mtime_ms>-<size>"`), `Accept-Ranges:
  bytes`, `Cache-Control`/`Content-Disposition` never set unless via
  `opts.headers`. Bodies are `body_bytes` for binary safety.
- **Directory index** — `/dir` (no trailing slash) → 301 to
  `/dir/`; `/dir/` serves `index.html` if present, else the opt-in
  listing, else 404.
- **HEAD** — same headers as GET, no body. The server layer
  (serve-epoll) also honors HEAD itself; both paths are covered.

## Layout

- `src/index.hnl` — the whole library (helpers + fs port + handler).
- `src/index.test.hnl` — `hnl test` suite (fake fs port injected;
  covers traversal, MIME, 304, 206/416, listing on/off, dotfiles,
  redirects, HEAD, 405, headers merge).
- `pkg.hnl` — package manifest (`http-static`, no deps).
- `scripts/hnl-gate.sh` — parse/typecheck/lint/test gate.
- `workflows/gate.yaml` — CI workflow (bupha shell runner).
- `ci.sh`, `bupha.toml`, `LICENSE` (MIT), `.gitignore`, `AGENTS.md`.

## Gate

```bash
# from this repo, with a dogfood hnl:
HANDL_EXT_DIR=<path-to-dogfood-debug> \
  <path-to-dogfood>/hnl test src/index.test.hnl

# or the full gate script (typecheck/lint where the dialect allows,
# plus tests):
HANDL_EXT_DIR=<path-to-dogfood-debug> bash scripts/hnl-gate.sh
```

Tests run against an injected fake `fs` port so they are hermetic;
the built-in `ffi` backend is exercised by a smoke test in
`src/index.test.hnl` when run on linux/x86_64.

## License

MIT — see `LICENSE`.
