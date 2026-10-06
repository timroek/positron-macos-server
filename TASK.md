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

(not yet run)
