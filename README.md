# Positron remote server for macOS (unofficial build)

Builds the Positron remote server ("REH") for macOS on Apple Silicon from the
unmodified Positron source, so that Positron's built-in Remote - SSH can connect
to a Mac. Posit publishes this server only for Linux.

**This is not an official build of Posit Software, PBC.** Positron is a
trademark of Posit Software, PBC. Positron is licensed under the Elastic
License 2.0 with the Positron Education License Rider; see `LICENSE.txt`, which
is also included in every build.

## Modifications

None. The workflow checks out `posit-dev/positron` at the requested commit and
runs Positron's own build target `vscode-reh-darwin-arm64`. Any change to the
Positron source needed to make the build work will be listed here, with what
was changed and why.
