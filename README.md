# Positron remote server for macOS (unofficial build)

Builds the Positron remote server ("REH") for macOS on Apple Silicon from the
unmodified Positron source, so that Positron's built-in Remote - SSH can connect
to a Mac. Posit publishes this server only for Linux.

**This is not an official build of Posit Software, PBC.** Positron is a
trademark of Posit Software, PBC. Positron is licensed under the Elastic
License 2.0 with the Positron Education License Rider; see `LICENSE.txt`, which
is also included in every build.

## Modifications

No changes to the Positron source. The workflow checks out
`posit-dev/positron` at the requested commit and runs Positron's own gulp
tasks. Any change to the Positron source needed to make the build work will be
listed here, with what was changed and why.

How the build differs from running `npm run gulp vscode-reh-darwin-arm64` in
one go:

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
