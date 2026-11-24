# libjpeg-turbo

Container images with the command-line tools of [libjpeg-turbo](https://libjpeg-turbo.org/). `jpegtran` rotates, flips, crops and optimizes JPEG files by rearranging their compressed data, so the image is never decoded and compressed again. `cjpeg` and `djpeg` convert between JPEG and PNG, BMP, GIF, PPM or Targa, `rdjpgcom` and `wrjpgcom` read and write JPEG comments, and `tjbench` benchmarks the TurboJPEG API. `latest` also includes ExifTool, to reset the Exif orientation of photos that `jpegtran` has turned upright.

The tools are compiled from the signed release tarball and linked statically against that release, so they do not use the older libjpeg-turbo that Ubuntu and Alpine ship. The images are rebuilt when libjpeg-turbo publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the libjpeg-turbo Project. Report problems with the image in this repository and problems with the tools themselves [upstream](https://github.com/libjpeg-turbo/libjpeg-turbo/issues).

## Quick start

Rotate a photo by 90 degrees clockwise without recompressing it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/libjpeg-turbo -rotate 90 -copy all -outfile rotated.jpg photo.jpg
```

The entrypoint runs `jpegtran` under `tini` in `/work`, so file names are relative to the directory you mount. Without `-outfile`, jpegtran writes to standard output, and without an input file it reads standard input, so a single file needs no mount:

```sh
docker run --rm -i ghcr.io/randomcontainers/libjpeg-turbo -rotate 180 < photo.jpg > rotated.jpg
```

Run the other tools with `--entrypoint`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint cjpeg \
  ghcr.io/randomcontainers/libjpeg-turbo -quality 85 -outfile photo.jpg photo.png

docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint djpeg \
  ghcr.io/randomcontainers/libjpeg-turbo -png -outfile photo.png photo.jpg
```

The tools take options with a single dash, and the names may be abbreviated. cjpeg, djpeg and jpegtran print their version with `-version`, and an option a tool does not know, such as `-help`, makes it print its usage. Upstream's [usage.txt](https://github.com/libjpeg-turbo/libjpeg-turbo/blob/main/doc/usage.txt) describes every option.

## Lossless changes with jpegtran

jpegtran moves whole blocks of compressed data, so the result has exactly the pixels of the original, turned or cut. It can `-rotate 90`, `180` or `270` degrees clockwise, `-flip horizontal` or `vertical`, `-transpose`, `-transverse`, `-crop WxH+X+Y` and drop the color with `-grayscale`. `-optimize` and `-progressive` make a file smaller without changing a pixel:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/libjpeg-turbo -copy none -optimize -progressive -outfile small.jpg photo.jpg
```

`-copy` chooses the metadata that goes into the new file: `none`, `comments` (the default), `icc` for the color profile alone, or `all`, which also keeps the Exif data. A photo tagged with a wide-gamut profile such as Display P3 is shown with the wrong colors without it, so use `-copy icc` rather than `-copy none` to strip such photos.

JPEG stores pixels in blocks that are usually 8 or 16 pixels on each side, depending on the chroma subsampling: 16x16 for 4:2:0, 16x8 for 4:2:2, and 8x8 for 4:4:4 and grayscale. When the width or height is not a multiple of the block size, the partial blocks along the right or bottom edge cannot be transformed and stay as they were, which shows as a strip along one edge of the result. `-trim` drops them, so the result is a few pixels smaller, and `-perfect` makes jpegtran fail instead. `-crop` moves the top left corner of the region up and left to a block boundary, so the result can be slightly larger than asked for.

`-outfile` may name the input file: jpegtran reads all of it before it writes. Add `-nooverwrite` to refuse to replace an existing file. Data after the end of the image, such as the video part of a motion photo, is not copied.

### Exif orientation

Cameras and phones often store a photo as the sensor saw it and record in the Exif Orientation tag how to turn it for display. jpegtran neither reads nor changes that tag. After turning the pixels with `-copy all`, the tag still tells viewers to rotate the photo, and they turn it a second time. The default image has ExifTool to reset the tag:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/libjpeg-turbo \
  -c 'jpegtran -rotate 90 -copy all -outfile upright.jpg photo.jpg && exiftool -overwrite_original -n -Orientation=1 upright.jpg'
```

`exiftool -n -Orientation photo.jpg` prints the tag. For 6 use `-rotate 90`, for 3 `-rotate 180` and for 8 `-rotate 270`. jpegtran copies the Exif thumbnail as it is, so programs that show the thumbnail still show it sideways; `exiftool -overwrite_original -ThumbnailImage= upright.jpg` removes it.

## Converting with cjpeg and djpeg

cjpeg reads PNG, BMP, GIF, PPM, PGM and Targa files and recognizes the format from the first byte. If it reports a Targa file as an unrecognized format, pass `-targa`. `-quality` sets the JPEG quality (75 by default), and `-progressive` and `-optimize` make the file smaller. It keeps the ICC profile of a PNG unless you pass `-noicc`. `-lossless 1` writes a lossless JPEG, which few other programs can open.

djpeg writes PPM or PGM by default, and PNG with `-png`, BMP with `-bmp`, GIF with `-gif` or Targa with `-targa`. `-scale 1/4` decodes a JPEG at a quarter of its size, much faster than decoding all of it and scaling it down.

## What is in the image

| | slim | default |
|---|---|---|
| `jpegtran`, `cjpeg`, `djpeg`, `rdjpgcom`, `wrjpgcom` and `tjbench`, linked statically against libjpeg-turbo | yes | yes |
| ExifTool, to read and fix metadata such as the Exif orientation | no | yes |

The tools load only the C and math libraries and the distro's zlib. The images have no libjpeg or TurboJPEG library and no headers, so they do not replace the distro's libjpeg for other programs. Not included: the man pages, upstream's TurboJPEG example programs (`tjcomp`, `tjdecomp`, `tjtran`) and the Java bindings. The build runs upstream's regression tests on every distro and platform before it installs anything. The CMake options and the output of `cjpeg -version` are in `/usr/local/share/randomcontainers/libjpeg-turbo/buildinfo`.

ExifTool in the default image is the [randomcontainers/exiftool](https://github.com/randomcontainers/exiftool) slim build, with the distro's Perl. The entrypoint runs `jpegtran`; run ExifTool with `--entrypoint exiftool`, or from a shell as in the example above.

## Default or slim

The default image (`latest`) adds ExifTool, for photos that record their orientation in Exif data: jpegtran turns the pixels and ExifTool resets the tag. `slim` has only the libjpeg-turbo tools. Use it when you only transform, convert or benchmark JPEG files, or as the base for your own image.

The default image is also published as `ghcr.io/randomcontainers/libjpeg-turbo-exiftool`, built in the [libjpeg-turbo-exiftool](https://github.com/randomcontainers/libjpeg-turbo-exiftool) repository with the same contents and a different digest.

## Tags

`<version>` is a libjpeg-turbo release such as `3.2.0`. `<minor>` and `<major>` are its shorter forms, `3.2` and `3`, and follow the newest release in that series.

| Default (with ExifTool) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current libjpeg-turbo version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation. On amd64 the tools use AVX2 when the CPU has it and SSE2 otherwise, and on arm64 they use Neon.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` follow new libjpeg-turbo releases and are rebuilt when the distro changes. The tools need no distro packages beyond the base image's, and `/usr/local/share/randomcontainers/libjpeg-turbo/runtime-deps` lists any they would need. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/libjpeg-turbo:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends optipng \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache optipng`. Packages that need libjpeg bring the distro's own, which the tools in `/usr/local/bin` do not load. The entrypoint is `["tini", "--", "jpegtran"]`; set your own `ENTRYPOINT` if your image runs a script. To pick up new libjpeg-turbo releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/libjpeg-turbo:latest \
  --repo randomcontainers/libjpeg-turbo --signer-repo randomcontainers/ci
```

Images published since November 2026 also have a keyless cosign signature from the same workflow, for admission controllers that check cosign signatures. Tags last built before then have only the attestation.

```sh
cosign verify ghcr.io/randomcontainers/libjpeg-turbo:latest \
  --certificate-identity https://github.com/randomcontainers/ci/.github/workflows/build.yml@refs/tags/v1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository randomcontainers/libjpeg-turbo
```

Images from `ghcr.io/randomcontainers/libjpeg-turbo-exiftool` are built in [randomcontainers/libjpeg-turbo-exiftool](https://github.com/randomcontainers/libjpeg-turbo-exiftool), so use `randomcontainers/libjpeg-turbo-exiftool` in `--repo` and `--certificate-github-workflow-repository`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/libjpeg-turbo:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the libjpeg-turbo Project's signing key in `keys/libjpeg-turbo.gpg` (fingerprint `0338 C8D8 D9FD A62C F9C4 21BD 7EC2 DBB6 F4DB F434`).

## Updates

The project checks the [libjpeg-turbo/libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo/releases) releases every 15 minutes. Betas are skipped. A release is picked up once it is 24 hours old and has the source tarball and its signature attached. The tarball is checked against the digest GitHub records for it, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new ExifTool image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t libjpeg-turbo:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile and test jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

This software is based in part on the work of the Independent JPEG Group.

The libjpeg code is under the IJG License and the TurboJPEG code under the Modified (3-clause) BSD License. The SIMD code is under the zlib License, and libspng, which cjpeg and djpeg use for PNG files, under the 2-clause BSD License and the PNG Reference Library License version 2. The image's license label is `IJG AND BSD-3-Clause AND Zlib AND BSD-2-Clause AND libpng-2.0`. The tools link the distro's zlib, which keeps its own license, and not the copy of zlib in the tarball.

The license files are in `/usr/local/share/randomcontainers/libjpeg-turbo/licenses/`: `LICENSE.md`, which explains how the licenses apply and has the text of the BSD License; `README.ijg`, with the IJG License; `spng-LICENSE` and `libpng-LICENSE.txt` from libspng; and `simd-NOTICE`, the zlib License notice of the SIMD code.

Every libjpeg-turbo version has a GitHub release in this repository, named `v<version>`, with the exact `libjpeg-turbo-<version>.tar.gz` that was compiled and its signature. The build applies no patches. `/usr/local/share/randomcontainers/libjpeg-turbo/source` lists that release and the upstream download URLs.

The default image also contains ExifTool, licensed under the same terms as Perl; see [randomcontainers/exiftool](https://github.com/randomcontainers/exiftool#licenses). The Ubuntu and Alpine packages in the images keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
