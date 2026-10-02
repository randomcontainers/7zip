# 7zip

Container images with [7-Zip](https://www.7-zip.org/), the file archiver, for creating, extracting and testing 7z, zip, tar, xz and other archives from the command line. `7zz`, the standalone console version of 7-Zip, is compiled from the source tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when a new 7-Zip version is released and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by Igor Pavlov, who develops 7-Zip. Report problems with the image in this repository and 7-Zip bugs at [sourceforge.net/p/sevenzip/bugs](https://sourceforge.net/p/sevenzip/bugs/).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/7zip \
  x archive.7z
```

Pack a directory into a 7z archive with maximum compression:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/7zip \
  a -mx=9 backup.7z project/
```

The entrypoint runs `7zz` under `tini` in `/work`, so file names are relative to the directory you mount. A few more commands:

```sh
# List the files in an archive
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/7zip \
  l archive.rar

# Test an archive without extracting it
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/7zip \
  t archive.zip

# Extract a disk image into the directory iso/
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/7zip \
  x -oiso image.iso

# Encrypt the contents and the file names; -p without a value asks for the password
docker run --rm -it --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/7zip \
  a -p -mhe=on private.7z documents/
```

Without arguments the image prints `7zz --help`, the list of commands and switches. The `i` command lists every format and codec in the build.

## What is in the image

- `7zz` in `/usr/local/bin`, and `7z` as a link to it. `7zz` has every format and codec compiled in and loads no plugins.
- `7zz` packs and unpacks 7z, zip, tar, xz, gzip, bzip2 and wim archives. It unpacks many more formats, among them RAR and RAR5, ISO, UDF, DMG, CAB, zstd, cpio, RPM, SquashFS and the VHD, VHDX, VMDK and QCOW2 disk images.
- The distro's C and C++ runtime libraries, the only libraries `7zz` links.

`7zz` is compiled from the C and C++ sources only. Upstream's optional assembler code, which makes some codecs faster, is not used on either architecture. The compiler flags are in `/usr/local/share/randomcontainers/7zip/buildinfo`.

## Default or slim

7-Zip's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a 7-Zip release such as `26.03`. `<major>` is its shorter form, `26`, and follows the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current 7-Zip version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

When 7-Zip adds files to an existing archive or deletes files from it, it writes the new archive next to the old one and then replaces the old one, so the directory needs room for both. When an archive stores Unix permissions, extracted files get them, limited by the umask and without the setuid, setgid and sticky bits.

## Untrusted archives

7-Zip parses archives in dozens of formats, and its releases often fix memory-safety bugs in those parsers, for example the heap overflow in the XZ decoder fixed in 26.02 (CVE-2026-14266). To fix CVE-2025-55188, 7-Zip 25.01 added security checks for the symbolic links it creates during extraction. The `-snld` switch relaxes them and `-snld20` turns them off, so pass neither for archives you did not create. For archives from unknown sources, also take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --network none --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges \
  --memory 2g \
  ghcr.io/randomcontainers/7zip x -ountrusted untrusted.zip
```

Mount only the directory the job needs. The images pick up a new 7-Zip release about a day after it is published.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new 7-Zip release and are rebuilt when the base image changes. `7zz` unpacks `.zst` files but cannot create them, so this example adds the `zstd` tool. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/7zip:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends zstd \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache zstd`. The entrypoint is `["tini", "--", "7zz"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new 7-Zip releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/7zip/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages 7-Zip needs at run time beyond those in the base image.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/7zip:latest \
  --repo randomcontainers/7zip --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/7zip:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the source tarball against the SHA-256 recorded in `package.yml`.

## Updates

The project checks the releases of [ip7z/7zip](https://github.com/ip7z/7zip/releases), where 7-zip.org publishes its downloads, every 15 minutes. A release is picked up once it is 24 hours old. 7-Zip publishes no checksum files or signatures, so the source tarball is checked against the SHA-256 digest GitHub records for the release asset. The new version and the tarball's SHA-256 are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t 7zip:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

7-Zip is licensed under the GNU Lesser General Public License, version 2.1 or later (LGPL-2.1-or-later). Some parts have other terms:

- BSD-3-Clause: the LZFSE and zstd decoders.
- BSD-2-Clause: the XXH64 hash code.
- The unRAR license restriction: the RAR decoders are under the LGPL with an extra condition from the unRAR license. The code may not be used to re-create the RAR compression algorithm, which is proprietary, or to develop a RAR (WinRAR) compatible archiver. The restriction has no SPDX identifier and appears in the license label as `LicenseRef-7-Zip-unRAR`.
- Public domain: many files, among them the LZMA code, state that they are in the public domain.

`License.txt`, which describes these terms, `copying.txt` (the LGPL) and `unRarLicense.txt` from the source tarball are in `/usr/local/share/randomcontainers/7zip/licenses/`. The image's license label is `LGPL-2.1-or-later AND BSD-2-Clause AND BSD-3-Clause AND LicenseRef-7-Zip-unRAR`. The Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- 7-Zip: every version has a GitHub release in this repository, named `v<version>`, with the exact source tarball that was compiled, for example `7z2603-src.tar.xz` for 26.03. The build applies no patches. The download URL is in `/usr/local/share/randomcontainers/7zip/source`.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every make option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
