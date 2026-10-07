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

(not yet run)
