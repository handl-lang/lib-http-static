# lib-http-static

`http-static` — static file serving for Handl HTTP servers, in pure
handl. The third library in the P-007 HTTP stack (after routing and
onion middleware): answers `GET`/`HEAD` for files under a configured
root and returns the `lib-http-rfc` response record, dropping straight
into `serve`/`serve-epoll` handlers and middleware chains.

## Layout

| path | role |
| --- | --- |
| `src/index.hnl` | the server — traversal-safe confinement, Content-Type by extension, Content-Length/Last-Modified/ETag (If-None-Match → 304), Accept-Ranges + Range → 206/416, HEAD, index.html, opt-in listing, 404/403 |
| `src/index.test.hnl` | tests |
| `pkg.hnl` | manifest |
| `scripts/hnl-gate.sh`, `ci.sh` | gate + bupha entry |
| `workflows/gate.yaml` | CI — shared `handl-gates` campaign |

## Contract

- Path traversal safety is the core invariant — confinement checks
  must stay ahead of any filesystem read.
- Pure `.hnl`, no Rust crate.

## Verification

```bash
HNL=../host-cli/target/debug/hnl scripts/hnl-gate.sh
```

<!-- myco:agent-awareness -->

## Agent awareness

- Session start / any work discussion: run `myco status --prompt`
  before proposing next steps — know the board first.
- User describes new work → offer to capture it (`myco todo add`,
  `issue new`, `proposal new`). Don't wait to be asked.
- Picking up a task: `myco resume <ID>` is the briefing.
- Wrapping up: `myco handoff <ID>` anything resumable; keep statuses
  honest (claimed → in-progress → blocked/done).

<!-- /myco:agent-awareness -->
