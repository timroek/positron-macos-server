# Security note: ark's loopback listeners accept connections from any local user

Draft for a possible report to Posit. Not filed.

## Affected

- ark (`posit-dev/ark`) at commit `5564f48` (Ark 0.1.252+266), as bundled with
  Positron 2026.09.1 build 2 (commit `467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2`).
  Other versions were not checked.
- Relevant on machines shared by several OS users, for example a Mac where one
  user's Positron connects over SSH to another user's account on the same
  machine.

## Issue

ark listens on `127.0.0.1` with no authentication and no check of who
connects. Any local user can connect to these listeners while an R session is
running:

| Listener | Code | What a connecting user can do |
| --- | --- | --- |
| DAP server | `crates/ark/src/dap/dap_server.rs`, `start_dap` | Accept loop, one client at a time. An `evaluate` request without `frameId` evaluates in the global environment (`dap_state.rs`, `frame_env`): **arbitrary R code runs as the session owner**. |
| LSP server | `crates/ark/src/lsp/backend.rs`, `start_lsp` | Accepts the first client. Can read session information (symbols, completions, hover of objects); if it connects first, the real client cannot connect. |
| Help proxy (actix-web, `HTTP/1.1`) | `crates/ark/src/help_proxy.rs` | `GET /dev-figure?file=<path>` returns the raw bytes of **any file the session owner can read** (the MIME type is derived from the extension, so a file needs a known extension). `GET /preview?file=<path>` renders any `.Rd` file. Other paths are forwarded to R's help server. |
| R's help server (`HTTP/1.0`) | R `tools::startDynamicHelp`, started by ark (`.ps.help.startOrReconnectToHelpServer`) | See below. Part of R, not ark. |

The five ZeroMQ Jupyter sockets are protected by the HMAC key in the
connection file and are not affected.

R's help server (R `src/library/tools/R/dynamicHelp.R`, `src/modules/internet/Rhttpd.c`)
also has no authentication. It serves, to any local user:

- files under the session's `tempdir()` via `/session/<path>`. `Rhttpd.c`
  removes `..` segments, so this stays inside the temp directory (symbolic
  links aside). The temp directory can hold plots, HTML output and data the
  session wrote there;
- help pages, vignettes, NEWS and other files of installed packages;
- `/library/<pkg>/Example/<topic>?local=FALSE` and `/library/<pkg>/Demo/<name>`,
  which run an installed package's example or demo code in the session, the
  former in the global environment;
- `/custom/<name>` handlers that loaded packages register.

## Fix applied in this repository

`patches/ark-peer-check.patch` adds a check right after `accept()` in the DAP
server, the LSP server and the help proxy: the client end of the connection
must be a TCP socket owned by a process of the same effective user as ark.
TCP has no `SO_PEERCRED` on macOS, so the user's processes and their sockets
are enumerated with libproc (`proc_listpids` with `PROC_UID_ONLY`,
`proc_pidinfo(PROC_PIDLISTFDS)`, `proc_pidfdinfo(PROC_PIDFDSOCKETINFO)`),
looking for a socket whose local address is the peer address and whose
remote address is the listener. On Linux the owner uid comes from
`/proc/net/tcp{,6}`. Any error rejects the connection. Rejected DAP and LSP
connections are closed and the server keeps waiting for the legitimate
client; the help proxy answers `403 Forbidden`.

R's help server is not patched (it is part of R). Setting `R_DISABLE_HTTPD=1`
in the session owner's environment (for example in `~/.Renviron`) stops R from
starting it; ark then also skips the help proxy, and the Help pane no longer
shows R help.

## Suggestions for upstream

- Authenticate the DAP and LSP connections, for example with a token passed
  through the Jupyter comm that announces the port, or listen on a Unix domain
  socket in a directory only the user can access. A per-connection owner check
  like the one above works without protocol changes.
- Restrict `/dev-figure` and `/preview` in the help proxy to the files the
  frontend actually requests, and authenticate the proxy.
- Serve R help in-process (calling `tools:::httpd()` from the proxy) instead
  of starting R's TCP help server, so the R help server is not reachable by
  other users.
