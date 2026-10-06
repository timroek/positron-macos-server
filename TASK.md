# Task for an agent: get a working macOS build of Positron's remote server

## Goal

Produce a working build of Positron's remote server ("REH") for macOS on Apple Silicon, using the GitHub Actions workflow in this repository (`.github/workflows/build.yml`).

## Background

- Positron (`posit-dev/positron`, a Code-OSS fork) ships its remote SSH server only for Linux. The server built here lets Positron's built-in "Open Remote - SSH" connect to a Mac (another user account on the same Mac).
- The target is Positron 2026.09.1, build 2, commit `467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2` (VS Code base 1.130.0). The server must be built from exactly this commit, because the client only accepts a server with the same commit.
- The client normally downloads the server via `serverDownloadUrlTemplate` in `product.json`: `https://cdn.posit.co/positron/${quality}/reh/${arch-long}/positron-reh-${os}-${arch}-${version}.tar.gz`. It installs it under `~/.positron-server/bin/<commit>/`.
- The workflow checks out `posit-dev/positron` at the commit, runs `npm install --prefix build`, `npm ci`, then `npm run gulp vscode-reh-darwin-arm64`, and uploads a tarball with `LICENSE.txt` included.
- Reference: Positron's own Linux equivalent, `.github/workflows/build-remote-ssh-linux.yml` in `posit-dev/positron`, for env vars and steps.

## Steps

1. Check the most recent workflow run first. One may already have finished; reuse its result instead of starting a new run if it tells you what you need.
2. Trigger the workflow with `gh workflow run build.yml`. If you cannot dispatch, add a temporary `push` trigger on `main` and remove it afterwards. Watch the run and read the logs.
3. On failure, diagnose and fix the workflow.
   - Prefer fixing the workflow (env vars, steps, tool versions) over modifying Positron's source.
   - If a source change is truly required, put it in a patch file in this repository and apply that patch during the workflow.
   - Document every change (what and why) under "Modifications" in `README.md`. This is a license requirement, see below.
4. Repeat until a run succeeds, or until you have strong evidence that it cannot work. In that case, stop and explain precisely why. Keep the number of runs reasonable: each macOS run may take 30–90 minutes.
5. When a run succeeds, inspect the artifact's structure and report it:
   - the top-level folder;
   - whether the node binary is present;
   - the `bin/` scripts, such as `bin/positron-server` or `code-server`, and `server.sh`;
   - the commit in `product.json`;
   - whether Positron's R support is included, such as the ark kernel and the `positron-r` extension.

## Constraints

- **Repository:** only touch this repository. Do not open issues or pull requests on `posit-dev/positron` or any other repository.
- **Public repository:** do not add secrets, tokens, personal data or local paths.
- **License:** Elastic License 2.0 with the Positron Education License Rider.
  - `LICENSE.txt` must ship with every copy.
  - Never remove Posit's copyright or licensing notices.
  - Modified copies must carry prominent notices of the modification.
- **Commits:** write commit messages in English and end each one with:

  ```
  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
  ```

- **Access:** if you cannot push or cannot read the Actions logs, stop immediately and report exactly which access is missing.

## Final report

Write the report concisely, in English. Append it to the end of this file under "Result", and give it in the session as well. Include:

- the outcome: success or failure;
- the URL of the successful run and the artifact name;
- the workflow changes made;
- any modifications to Positron's source;
- your findings on the artifact's structure;
- anything the owner must do.

## Result

**Outcome: success.** Run: https://github.com/timroek/positron-macos-server/actions/runs/37535699882
(run 5, commit `ad47fd7`). Artifact: `positron-reh-darwin-arm64-467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2`
(746 MB zip containing the `.tar.gz`; 2.3 GB unpacked; kept for 30 days).

### Runs

1. 37514052540: cancelled on purpose (before this task).
2. 37515139613: `npm run gulp vscode-reh-darwin-arm64` on `macos-15` ran out of
   JavaScript heap (8 GB limit; the runner has 7 GB RAM) in the emit of
   `compile-src`.
3. 37525014733: new Linux compile job failed in `npm ci` (`kerberos` needs
   `gssapi/gssapi.h`).
4. 37529064236: compile and extension builds passed; `copy-extension-binaries`
   failed as a standalone gulp task ("did not complete").
5. 37535699882: success.

### Workflow changes (`.github/workflows/build.yml`)

- New `compile` job on `ubuntu-24.04` (16 GB RAM plus 12 GB extra swap,
  Linux build headers installed) runs `compile-build-without-mangling` with a
  12 GB heap and uploads `out-build/` (platform-independent JavaScript).
- The `macos-15` job restores `out-build/`, then runs the remaining steps of
  Positron's `vscode-reh-darwin-arm64-min` task one by one:
  `compile-non-native-extensions-build`, `compile-copilot-extension-build`,
  `compile-extension-media-build`, Positron's `copyExtensionBinaries()` called
  directly from Node, `minify-vscode-reh`, `vscode-reh-darwin-arm64-min-ci`.
  As a result the server code is minified (not mangled).
- Packaging unchanged: `LICENSE.txt` and `BUILD-NOTICE.md` (this README) are
  added to the tarball.
- New `.github/workflows/inspect.yml` prints the structure of a build artifact.
- Documented under "Modifications" in `README.md`.

### Modifications to Positron's source

None.

### Artifact structure

- Top-level folder: `vscode-reh-darwin-arm64/` with `LICENSE.txt` (Posit
  copyright, Elastic License 2.0), `BUILD-NOTICE.md`, `NOTICE`, `node`,
  `bin/`, `extensions/` (68 entries), `node_modules/`, `out/`, `quarto/`,
  `resources/`, `package.json`, `product.json`.
- `node`: present, Mach-O 64-bit arm64.
- `bin/`: `positron-server` (runs `node out/server-main.js`),
  `remote-cli/positron`, `helpers/browser.sh`. There is no `code-server` and
  no top-level `server.sh`.
- `product.json`: `commit` `467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2`,
  `positronVersion` 2026.09.1, `positronBuildNumber` 2, `version` 1.130.0,
  `serverApplicationName` `positron-server`, `serverDataFolderName`
  `.positron-server`, `quality` `dailies`.
- R support: included. `extensions/positron-r` (with `dist/`) and
  `extensions/positron-r/resources/ark/ark` (Mach-O arm64). Also present:
  `positron-supervisor` with `kcserver` (arm64), `positron-python` with `pet`
  (universal), `positron-reticulate`.
- Native modules in `node_modules` (node-pty, @parcel/watcher, spdlog,
  sqlite3, ...) are arm64.

### What the owner must do

- Download the artifact from the run page (it expires after 30 days) and
  install it as `~/.positron-server/bin/467ff390dd8b4ffc8972ebb84d9e05a1fd5a4fe2/`
  (the contents of `vscode-reh-darwin-arm64/`), or serve it under the
  `serverDownloadUrlTemplate` name `positron-reh-darwin-arm64-<version>.tar.gz`.
- Check that `quality` (`dailies` in the source build) is acceptable to the
  installed app; the client matches on the commit, but compare with the
  app's own `product.json` if the connection is refused.
- Test a real connection from Positron's Remote - SSH; this was not tested
  here.
