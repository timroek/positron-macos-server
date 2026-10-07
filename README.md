# Positron remote server for macOS (unofficial build)

Builds the Positron remote server ("REH") for macOS on Apple Silicon from the
unmodified Positron source, so that Positron's built-in Remote - SSH can connect
to a Mac. Posit publishes this server only for Linux.

**This is not an official build of Posit Software, PBC.** Positron is a
trademark of Posit Software, PBC. Positron is licensed under the Elastic
License 2.0 with the Positron Education License Rider; see `LICENSE.txt`, which
is also included in every build.

## Modifications

The Positron source itself is not changed. The workflow checks out
`posit-dev/positron` at the requested commit and runs Positron's own gulp
tasks. The R kernel **ark**, which Positron includes as a git submodule
(`extensions/positron-r/ark`, `posit-dev/ark`, MIT licence), is modified as
described below.

### ark: listeners only accept connections from the same user

File: `patches/ark-peer-check.patch`, applied to ark commit `5564f48`
(Ark 0.1.252+266) before ark is built from source on the macOS runner. The
built binary replaces the prebuilt `extensions/positron-r/resources/ark/ark`
and is signed ad hoc (`codesign -s -`).

Why: ark opens a DAP server, an LSP server and an HTTP help proxy on
`127.0.0.1` without authentication. On a Mac with several user accounts, any
local user could connect to them. Through the DAP server they could run
R code as the user who owns the R session, and through the help proxy they
could read that user's files. See `SECURITY-NOTE.md`.

What changed:

- New module `crates/ark/src/peer_check.rs`. After `accept()`, it checks that
  the client end of the connection is a TCP socket owned by a process of the
  same user as ark. On macOS it enumerates the user's processes and their
  sockets with libproc (`proc_listpids`, `proc_pidinfo(PROC_PIDLISTFDS)`,
  `proc_pidfdinfo(PROC_PIDFDSOCKETINFO)`). On Linux it reads the socket owner
  from `/proc/net/tcp` and `/proc/net/tcp6`. Any error rejects the
  connection. Other platforms keep the original behaviour.
- DAP server (`crates/ark/src/dap/dap_server.rs`): connections that fail the
  check are closed and logged; the accept loop continues, so the legitimate
  client can still reconnect.
- LSP server (`crates/ark/src/lsp/backend.rs`): keeps accepting until a
  connection passes the check, instead of serving the first connection.
- Help proxy (`crates/ark/src/help_proxy.rs`): the check runs once per
  connection, and every request on a connection that fails it gets
  `403 Forbidden`.
- Tests: unit tests for the check and the help proxy, and CI tests with real
  connections from another process: accepted from the same user, rejected
  from the `nobody` user (for the help proxy: `200` versus `403`).

ark's `LICENSE` and the patch are included in the build under
`licenses/ark/`. All copyright and licence notices are kept.

### No AI components in the server

Removed from the server before packaging:

- `extensions/copilot`: the built-in GitHub Copilot Chat extension;
- `extensions/next-edit-suggestions`: Posit AI next-edit suggestions;
- `node_modules/@github/copilot*`: the GitHub Copilot packages (such as
  `@github/copilot-darwin-arm64`), used only by the agent host
  (`out/vs/platform/agentHost/node/agentHostMain.js`). Disable the agent host
  in Positron with `"chat.agentHost.enabled": false`.

The workflow fails if any of these, or any extension or `node_modules`
package whose name contains "copilot", remains. It then starts the server
with a temporary connection token and checks that it listens, so nothing the
server needs at startup was removed.

Reason: in the setup this build is made for, every component in the server
runs as the user who owns sensitive data, and no AI component may run on that
side.

### Build

How the build differs from running `npm run gulp vscode-reh-darwin-arm64` in
one go:

- ark is patched and built from source (see above).

- The TypeScript compile of `src/` (`compile-build-without-mangling`) runs on
  a Linux runner with a 12 GB JavaScript heap, and its output (`out-build/`)
  is passed to the macOS job. On the macOS arm64 runner (7 GB RAM) this step
  ran out of memory at Positron's default 8 GB heap. The output is plain
  JavaScript and does not depend on the platform.
- The macOS job runs the remaining steps of Positron's
  `vscode-reh-darwin-arm64-min` task one by one: building the extensions,
  bundling and minifying the server (`minify-vscode-reh`) and packaging it for
  darwin-arm64 (`vscode-reh-darwin-arm64-min-ci`). The server code is
  therefore minified. It is not mangled, the same as Positron's non-minified
  target.
