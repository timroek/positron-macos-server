# Task for an agent: harden ark's local TCP listeners and ship it in the macOS server

## Context

The macOS remote server built by this repository (see `TASK.md`) is used for one purpose. Positron runs as user A and connects over SSH to a second, hidden macOS user B on the same Mac. B owns sensitive files that A must never be able to read. The OS user boundary protects those files. The goal of this task is that nothing in the server lets a process of user A run code as user B.

The R kernel ark, which is bundled in the server at `extensions/positron-r/resources/ark/ark`, opens several TCP listeners on `127.0.0.1`. Any local user can reach them. Observed on a running session:

- 5 ZeroMQ Jupyter sockets. These are protected by the HMAC key in the connection file, which only B can read, so they are out of scope.
- **The DAP server** (`crates/ark/src/dap/dap_server.rs`, `start_dap`). It has no authentication. It accepts clients in a loop, one at a time. An `evaluate` request without a `frameId` evaluates in `R_ENVS.global` (`dap_state.rs`, `frame_env`), so any local process that gets served can run arbitrary R code as B.
- **The LSP server** (`crates/ark/src/lsp/backend.rs`, `start_lsp`). It has no authentication and accepts a single client.
- Two HTTP listeners. One answered `HTTP/1.1 400` with a `date` header; that is probably a Rust/hyper server inside ark. The other answered `HTTP/1.0 400`; that is possibly R's own dynamic help server (`tools::startDynamicHelp`).

Bundled ark version: `Ark 0.1.252+266.5564f48`, which is commit `5564f48` of `posit-dev/ark`. Verify this against the server build of Positron commit `467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2`.

## Goal

A server build, as in `TASK.md`, whose ark rejects every TCP connection to its DAP and LSP listeners that does not come from a process owned by the same user as ark itself. Do the same for any other TCP listener ark opens that can execute code or serve session data.

## Approach (suggested; improve it if you find something better)

1. **Patch ark at the bundled commit, minimally.**
   - Right after `accept()`, determine whether the peer socket (`127.0.0.1:<peer port>`) belongs to a process with the same uid as ark.
   - macOS has no `SO_PEERCRED` for TCP. Instead, enumerate the current user's processes with libproc: `proc_listpids` filtered on uid, `proc_pidinfo` with `PROC_PIDLISTFDS`, and `proc_pidfdinfo` with `PROC_PIDFDSOCKETINFO`. Look for a TCP socket whose local address is `127.0.0.1:<peer port>` and whose remote port is ark's listening port.
   - If no match is found, close the connection immediately and log it. Fail closed on any error.
   - Apply this to DAP and LSP. For DAP, keep the accept loop working for legitimate reconnects.
2. **Find out what the two HTTP listeners are.**
   - Find out whether they can execute code or serve files from the R session, such as the session temp directory, plots or HTML output, to any local user.
   - If ark owns the listener, apply the same peer check.
   - If it is R's own help server, report precisely what it serves and to whom, and propose a mitigation. Do not patch R itself.
3. **Tests.**
   - Unit-test the peer-ownership check: a connection from the same process must be accepted, and an unknown peer port must be rejected.
   - If feasible in CI, test a real rejection by connecting as a different user on the macOS runner (for example with `sudo -u nobody`).
4. **Build and ship.**
   - Add a job or steps to `.github/workflows/build.yml`.
   - Build the patched ark for `aarch64-apple-darwin` on a macOS runner, replace `extensions/positron-r/resources/ark/ark` in the server tarball, and sign it ad hoc (`codesign -s -`).
   - Keep the patch as a `.patch` file in this repository, applied during the workflow.
   - Produce a new artifact the same way as before.
5. **Documentation and licenses.**
   - ark is MIT-licensed, while Positron is under the Elastic License 2.0.
   - Under "Modifications" in `README.md`, document what was changed in ark and why.
   - Keep all licence and copyright notices intact.
6. **Write-up for Posit.**
   - Write `SECURITY-NOTE.md`: a short, factual description of the issue and the fix, which the owner may decide to report to Posit.
   - Do not file it yourself.

## Addition: leave out GitHub Copilot

The server build must leave out the built-in GitHub Copilot Chat extension.
Remove the folder `extensions/copilot` from `vscode-reh-darwin-arm64/` before
packaging, and make the workflow fail if any extension folder whose name
contains "copilot" remains. Reason: in the owner's setup every extension in the
server runs as a user that has access to sensitive data, and no AI component may
run on that side. Record this under "Modifications" in `README.md`.

## Addition 2: remove the remaining AI components

The owner decided that the server build must also leave out:

- `node_modules/@github/copilot-darwin-arm64` and any other
  `node_modules/@github/copilot*` package. They are only referenced by the agent
  host (`out/vs/platform/agentHost/node/agentHostMain.js`), which the owner
  disables with `chat.agentHost.enabled: false`;
- the built-in extension `extensions/next-edit-suggestions` (Posit AI next-edit
  suggestions).

Extend the post-packaging check so the build fails if any of these, or any
extension or module whose name contains "copilot", remains. Verify in the
workflow that the server still starts after the removal. Document both
removals and the reason under "Modifications" in `README.md`. Reason: every
component in this server runs as the user that owns sensitive data, and no AI
component may run on that side.

## Constraints

- **Repository:** only touch this repository. Do not open issues or pull requests on `posit-dev/ark`, `posit-dev/positron` or any other repository.
- **Public repository:** no secrets, tokens, personal data or local paths.
- **Runs:** keep the number of runs reasonable.
- **Commits:** write commit messages in English and end each one with:

  ```
  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
  ```

- **Access:** if you lack access to push or to read the Actions logs, stop immediately and report exactly what is missing.

## Final report

Append the report at the end of this file under "Result" and push it. Include:

- the outcome;
- the URL of the successful run and the artifact name;
- the patch, summarised;
- the test results;
- your findings on both HTTP listeners;
- any remaining exposure;
- anything the owner must do.

## Result

**Outcome: success.** Run 9: https://github.com/timroek/positron-macos-server/actions/runs/37605720596
(commit `a4cc57e`). Artifact: `positron-reh-darwin-arm64-467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2`
(631 MB, kept until 2026-11-06). The earlier artifact of `TASK.md` (run 5) has
the unpatched ark and the AI components; do not use it.

Runs for this task: 6 and 7 cancelled on purpose (superseded by the Copilot
additions), 8 failed in the new AI check (it found more Copilot modules, see
below), 9 succeeded.

### Version check

Positron `467ff390` pins the ark submodule at `5564f488e539cfdd344f583a7aeb40250f7d20d3`;
`crates/ark/Cargo.toml` says 0.1.252 and the commit is 266 commits after tag
`0.1.252`, so the bundled ark is `0.1.252+266.5564f48` as stated.

### The patch (`patches/ark-peer-check.patch`)

- New `crates/ark/src/peer_check.rs`: `is_same_user_peer(server, local, peer)`
  looks for the client end of the connection, a TCP socket with local address
  `peer` and remote address `local`, among the sockets of processes owned by
  ark's effective uid. macOS: `proc_listpids(PROC_UID_ONLY, euid)`,
  `proc_pidinfo(PROC_PIDLISTFDS)`, `proc_pidfdinfo(PROC_PIDFDSOCKETINFO)`
  (the `socket_fdinfo` structs are declared `#[repr(C)]` from
  `<sys/proc_info.h>`, since `libc` lacks them). Linux: owner uid from
  `/proc/net/tcp{,6}`. No match or any error: reject and log. Other platforms:
  unchanged behaviour.
- DAP (`dap_server.rs`): rejected connections are closed and the accept loop
  continues, so reconnects keep working.
- LSP (`lsp/backend.rs`): accepts in a loop until a connection passes the
  check (previously the first connection was served, so another user could
  also block the real client).
- Help proxy (`help_proxy.rs`): `HttpServer::on_connect` runs the check once
  per connection; a middleware answers `403` to every request on a connection
  that failed or was never checked.
- ark is built from source on the macOS runner (`cargo build --release`,
  `ARK_BUILD_VERSION=0.1.252+266`, so it reports the same version), signed ad
  hoc; Positron's install script picks up `target/release/ark` ("Using locally
  built Ark"). The packaging step checks that the shipped
  `extensions/positron-r/resources/ark/ark` is byte-identical to that build,
  re-signs it and verifies the signature (`valid on disk`). ark's `LICENSE`
  and the patch ship under `licenses/ark/`.

### Tests (all passed on macOS in run 9, and on Linux locally)

- Unit: `test_accepts_connection_from_same_process`,
  `test_rejects_unknown_peer_port` (unknown peer port and wrong listener port
  both rejected), `test_help_proxy_serves_same_user` (not 403), and on Linux
  `test_linux_find_owner`.
- Real connections from another process: same user (`runner`) accepted,
  `sudo -u nobody` rejected ("Connection from nobody: rejected as expected").
- Help proxy end to end: `GET /dev-figure?file=<probe>.txt` returned `200` to
  the same user and `403` to `nobody`. The `200` also demonstrates the
  original issue: without the patch any local user could read files through
  this endpoint.

### The two HTTP listeners

1. `HTTP/1.1` with a `date` header: **ark's help proxy** (actix-web,
   `help_proxy.rs`). Besides proxying to R's help server it has
   `/dev-figure?file=<path>` (returns the raw bytes of any file the session
   owner can read, MIME type from the extension) and `/preview?file=<path>`
   (renders any `.Rd` file through R). Now protected by the peer check.
2. `HTTP/1.0`: **R's own dynamic help server** (`tools::startDynamicHelp`,
   started by ark's `.ps.help.startOrReconnectToHelpServer`; R
   `dynamicHelp.R` and `Rhttpd.c`). Not patched (part of R). To any local
   user it serves: files under the session `tempdir()` via `/session/<path>`
   (`..` segments are removed, so it stays in the temp dir apart from
   symlinks); help pages, vignettes, NEWS, DESCRIPTION and other files of
   installed packages; it **runs installed packages' example code**
   (`/library/<pkg>/Example/<topic>?local=FALSE` evaluates in the global
   environment) and demos (`/library/<pkg>/Demo/<name>`); and it calls
   `/custom/<name>` handlers registered by loaded packages. Mitigation:
   `R_DISABLE_HTTPD=1` in user B's environment (for example `~/.Renviron`).
   R then does not start the server, ark skips the proxy too, and the Help
   pane shows no R help. Longer term ark could serve help in-process
   (`tools:::httpd()`) without a TCP server (see `SECURITY-NOTE.md`).

### AI components removed (additions 1 and 2)

- Removed: `extensions/copilot`, `extensions/next-edit-suggestions`, and
  every `node_modules` package with "copilot" in its name:
  `@github/copilot`, `@github/copilot-sdk`, `@github/copilot-darwin-arm64`,
  `@vscode/copilot-api`, and `@github/copilot` and `@github/copilot-sdk`
  nested under `node_modules/ai-provider-bridge/`.
- Core files referencing them: `@github/copilot` and `@vscode/copilot-api`
  only in `out/vs/platform/agentHost/node/agentHostMain.js`.
- The build fails if any of them, or any extension or module with "copilot"
  in its name, remains (run 8 failed on exactly this before the removal was
  widened).
- The server still starts: `positron-server --help` works, and started with
  a temporary token it logged "Extension host agent listening on 50192" and
  "Extension host agent started", answered `/version` with `200`, and showed
  no missing-module errors.

### Remaining exposure

- R's own help server (above), unless `R_DISABLE_HTTPD=1` is set.
- `node_modules/ai-provider-bridge` (Posit's AI provider bridge from the
  `ai-lib` submodule) remains: it has no "copilot" in its name and
  `out/server-main.js` references it, so removing it could break the server
  at startup. The owner should decide; it would need a new start test.
- Not inspected: `positron-supervisor`'s `kcserver` (kernel supervisor) and
  other extensions' listeners, Python's kernel and its tools, and the
  Jupyter sockets (protected by the HMAC key, as stated in the task).
- The peer check trusts any process of the same uid; that is the intended
  boundary.
- Windows and other non-macOS/Linux platforms are unchanged (not relevant
  here).

### Artifact check

Inspection run https://github.com/timroek/positron-macos-server/actions/runs/37612898808
(`inspect.yml`) on the artifact: 1.9 GB unpacked; `extensions/` has 66
extensions, without `copilot` and `next-edit-suggestions`; `licenses/ark/`
present; `extensions/positron-r/resources/ark/ark` is Mach-O arm64; commit
in `product.json` is `467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2`.

### What the owner must do

- Use the new artifact from run 9 and replace the server installed from
  run 5 under `~/.positron-server/bin/467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2/`.
- Set `"chat.agentHost.enabled": false` in Positron, since the agent host's
  packages are removed.
- Decide on `R_DISABLE_HTTPD=1` for user B (closes R's help server, loses the
  R Help pane).
- Decide on `ai-provider-bridge`.
- Decide whether to send `SECURITY-NOTE.md` to Posit (not filed).
- Test a real session: R console, LSP features (completion), the debugger
  and the Help pane, to confirm the legitimate clients pass the check.
