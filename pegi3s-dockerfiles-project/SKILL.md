---
name: pegi3s-dockerfiles-project
description: >
  Manage Dockerfiles and building Docker images for the pegi3S Bioinformatics Docker Images project.
compatibility: opencode
---

# When to use this skill

Load this skill when you are:

- Updating Parametrized Docker Images

---

# Updating Parametrized Docker Images

This document describes the full process for updating a Docker image whose `Dockerfile` uses `ARG VERSION` and whose build is driven from a `BUILD.md` file.

Examples in this repository include `seqkit/2.x`, `seqtk/1.x`, and `bedtools/2.x`.

## Goal

When updating one of these images to a new software release, the process must:

1. Confirm the upstream version exists.
2. Update the local `BUILD.md` instructions.
3. Build the Docker image using the command described in `BUILD.md`.
4. Verify the built image reports the expected version.
5. Run the automated `bdip test-images` check when the image has `auto_tests`.
6. Update `metadata/metadata.json` only if build, verification, and tests succeeded.

## 1. Locate the target image

Identify the directory for the software image and confirm it contains at least:

1. `Dockerfile`
2. `BUILD.md`
3. Usually `README.md`

Typical examples:

1. `seqkit/2.x/`
2. `seqtk/1.x/`
3. `bedtools/2.x/`

## 2. Confirm the Dockerfile is parametrized

Open the `Dockerfile` and confirm it has:

1. `ARG VERSION`
2. Download or build logic that uses `${VERSION}`

Example pattern:

```dockerfile
ARG VERSION

RUN wget "https://example.org/tool/v${VERSION}/tool-${VERSION}.tar.gz"
```

If the Dockerfile is not parametrized, this guide does not fully apply and the Dockerfile itself may need version-specific edits.

## 3. Confirm the upstream release exists

Before changing the repository, confirm the requested version is available from the upstream source used in the `Dockerfile`.

Check one of these:

1. The upstream GitHub release page.
2. The exact download URL pattern used by the `Dockerfile`.
3. The upstream source or release directory.

Do not update metadata for a version that does not exist upstream.

## 4. Review the current metadata entry

Open `metadata/metadata.json` and find the object for the tool by its `name` field.

Review these fields before editing:

1. `recommended`
2. `latest`
3. `no_longer_tested`
4. `not_working`
5. `auto_tests`

Pay special attention to whether `auto_tests[].docker_image` uses:

1. a versioned image tag such as `pegi3s/bedtools:2.31.1`, or
2. an unversioned image such as `pegi3s/seqkit`

Only versioned references need to be updated.

## 5. Update `BUILD.md`

Edit the image directory `BUILD.md` file.

Update:

1. The version variable in the example build command.
2. The build log with the new version.
3. Any incorrect wording found while editing.

Example:

```bash
seqkit_version=2.13.0 && docker build ./ -t pegi3s/seqkit:${seqkit_version} --build-arg VERSION=${seqkit_version} && docker tag pegi3s/seqkit:${seqkit_version} pegi3s/seqkit:latest
```

Add a build log line in the existing format:

```text
- 2.13.0 - DD/MM/YYYY - Hugo Lopez Fernandez
```

Use the repository's current naming and date format already present in that file.

Build log entries must be ordered from newest to oldest (newest version first). Always add new entries at the top of the build log, and verify the existing entries are in correct order.

## 6. Build the image exactly as described in `BUILD.md`

Run the build command from the image directory itself.

Examples:

```bash
docker build ./ -t pegi3s/seqkit:2.13.0 --build-arg VERSION=2.13.0 && docker tag pegi3s/seqkit:2.13.0 pegi3s/seqkit:latest
```

```bash
docker build ./ -t pegi3s/seqtk:1.5 --build-arg VERSION=1.5 && docker tag pegi3s/seqtk:1.5 pegi3s/seqtk:latest
```

```bash
docker build ./ -t pegi3s/bedtools:2.31.1 --build-arg VERSION=2.31.1 && docker tag pegi3s/bedtools:2.31.1 pegi3s/bedtools:latest
```

Important rules:

1. Use the command documented in that image's `BUILD.md`.
2. Run it from the image directory.
3. Keep the version tag and `latest` tag behavior unchanged unless there is a clear repo-wide reason to change it.

## 7. Verify the built image

After the build finishes, confirm the container reports the expected version.

Use the correct invocation for the image's entrypoint behavior.

Examples:

1. Images with an entrypoint that is the tool itself:

```bash
docker run --rm pegi3s/seqkit:2.13.0 version
```

```bash
docker run --rm pegi3s/seqtk:1.5
```

2. Images without a tool entrypoint may require calling the binary explicitly:

```bash
docker run --rm pegi3s/bedtools:2.31.1 bedtools --version
```

The expected output must match the requested version.

If the build succeeds but the version check fails, do not update `metadata/metadata.json` yet.

## 8. Run `bdip test-images` when auto-tests exist

If the tool has `auto_tests` in `metadata/metadata.json`, run the repository test flow before updating metadata.

Use the `bdip-tools` conda environment:

```bash
source /home/hlfernandez/miniconda3/etc/profile.d/conda.sh && conda activate bdip-tools
```

Run only the target test with `--image`, **always with `--dry-run`**:

```bash
bdip test-images --image htseq --skip-pull --dry-run
```

Important notes:

1. Run the command from the repository root.
2. `--image` should match the metadata `name` field such as `htseq`, not necessarily the Docker repository string such as `pegi3s/htseq`.
3. Use `--skip-pull` when you want to test the image you just built locally.
4. **Always use `--dry-run`.** Running `bdip test-images` **without** `--dry-run` makes `bdip` rewrite the entire `metadata/metadata.json` using Python `json.dump` defaults, which re-escapes non-ASCII characters as `\uXXXX` (e.g. `GATK 3⁠` → `GATK 3\u2060`, `Schröder` → `Schr\u00f6der`), expands single-element arrays onto multiple lines, and drops the trailing newline — producing a large unintended diff. `--dry-run` runs the test but does not write metadata, so metadata is changed only deliberately by the manual edit in Step 10.
5. If the auto-test uses a versioned `docker_image`, update that field in `metadata/metadata.json` (Step 10) so the test exercises the new image.
6. If the test definition is broken, fix the metadata test entry before considering the image update complete.

Example:

```bash
source /home/hlfernandez/miniconda3/etc/profile.d/conda.sh && conda activate bdip-tools && bdip test-images --image htseq --skip-pull --dry-run
```

Only proceed to the final metadata update (Step 10) after the relevant `bdip` test passes.

## 9. Handle failures before metadata changes

If the build or verification fails:

1. Inspect whether the upstream archive layout changed.
2. Inspect whether the command used to report the version is wrong for that image.
3. Inspect whether the Dockerfile contains brittle commands such as broad `mv` patterns.
4. Fix the image definition if needed.
5. Rebuild and re-verify.

Only proceed to metadata after the image is confirmed to build and report the expected version.

## 10. Update `metadata/metadata.json`

Once the image is confirmed to work, update the tool object in `metadata/metadata.json`.

Because metadata is edited manually (not rewritten by `bdip`), only touch the fields listed below and preserve the file's existing formatting conventions: literal UTF-8 characters, 2-space indentation, inline single-element arrays, and a trailing newline at the end of the file.

### Required changes

1. Set `recommended[0].version` to the new version.
2. Set `recommended[0].date` to the test date in `DD/MM/YYYY` format.
3. Set `latest` to the new version.
4. Move the previously recommended or latest version into `no_longer_tested`.

### Conditional changes

1. If `auto_tests[].docker_image` contains an explicit version tag, update it to the new version.
2. If `auto_tests[].docker_image` is unversioned, leave it unchanged.

### Do not change these unless required

1. `not_working`
2. `bug_found`
3. command examples
4. URLs

Only touch those fields if the update actually requires it.

## 11. How to move the previous version

When a new version becomes the tested recommended version:

1. Take the old `latest` value.
2. Add it to `no_longer_tested` at the front (newest first).
3. Do not remove older historical values already present there.
4. Ensure the entire `no_longer_tested` array is ordered from newest to oldest (newest version first). When adding a new version, insert it at the front rather than appending at the end, and check that existing entries are in correct order.

Example transition:

1. `latest`: `2.12.0` -> `2.13.0`
2. add `2.12.0` to `no_longer_tested`

## 12. Update date policy

When the update succeeded and metadata is being changed, use the date of the successful test/build in `recommended[0].date`.

Format:

```text
DD/MM/YYYY
```

Example:

```text
07/07/2026
```

## 13. Final checks

Before considering the update complete, confirm:

1. `BUILD.md` points to the new version.
2. The build log in `BUILD.md` contains the new version.
3. The Docker image builds successfully.
4. The Docker image reports the expected version.
5. If `auto_tests` exist, the `bdip test-images --dry-run` run passed.
6. `metadata/metadata.json` has the new `recommended` and `latest` values.
7. The previous version was moved to `no_longer_tested` at the front (newest first).
8. The build log in `BUILD.md` is ordered from newest to oldest (newest first).
9. The `no_longer_tested` array is ordered from newest to oldest (newest first).
10. Any versioned `auto_tests[].docker_image` reference was updated.

## 14. Minimal checklist

Use this checklist for each parametrized image update:

1. Find image directory and `BUILD.md`.
2. Confirm `ARG VERSION` exists in `Dockerfile`.
3. Confirm the upstream release exists.
4. Update the `BUILD.md` version and build log.
5. Run the `docker build` command from `BUILD.md`.
6. Verify the built image version with `docker run`.
7. If `auto_tests` exist, run `bdip test-images --dry-run` for that tool.
8. Update `metadata/metadata.json` only if verification and tests succeeded.
9. Set `recommended[0].date` to the successful test date.
10. Move the old version into `no_longer_tested` at the front (newest first), ensuring the array stays ordered newest-to-oldest.
11. Update versioned auto-test image tags if present.
12. Verify build log and `no_longer_tested` are both ordered newest-to-oldest.

## 15. Example outcomes

For a successful update like the ones performed for `seqkit`, `seqtk`, and `bedtools`, the end state should be:

1. `BUILD.md` updated to the new target version.
2. Local Docker image built and tagged with both the explicit version and `latest`.
3. Version check output matches the requested version.
4. If `auto_tests` exist, the relevant `bdip test-images` run passes.
5. `metadata/metadata.json` updated so the new version is both `latest` and `recommended`.
6. The previous tested version moved into `no_longer_tested`.
