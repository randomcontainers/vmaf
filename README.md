# vmaf

Container images with `vmaf`, the command-line tool of [VMAF](https://github.com/Netflix/vmaf) by Netflix. VMAF (Video Multi-Method Assessment Fusion) predicts how viewers would rate the quality of an encoded video. `vmaf` compares the encode with its source frame by frame and writes a score for each frame and for the whole video, and on request PSNR, SSIM, MS-SSIM, CIEDE2000 and the CAMBI banding score. The default image also contains FFmpeg, which decodes encoded videos into the uncompressed input that vmaf reads. The images are rebuilt when VMAF publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by Netflix. Report problems with the image in this repository and problems with VMAF itself [upstream](https://github.com/Netflix/vmaf/issues).

## Quick start

vmaf reads Y4M and raw YUV video only. To score an encode, decode it and its source to Y4M with FFmpeg from the default image, then run vmaf on the two files:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/vmaf -c '
    ffmpeg -loglevel error -y -i source.mp4 -pix_fmt yuv420p -f yuv4mpegpipe reference.y4m &&
    ffmpeg -loglevel error -y -i encoded.mp4 -pix_fmt yuv420p -f yuv4mpegpipe distorted.y4m &&
    vmaf -r reference.y4m -d distorted.y4m --json -o vmaf.json'
```

`-pix_fmt yuv420p` converts both videos to 8 bits. For 10-bit videos, such as 10-bit AV1 and HEVC encodes, use `-pix_fmt yuv420p10le` in both `ffmpeg` commands so that vmaf compares them at 10 bits. Upstream recommends 10 bits for the VMAF v1 models even when the encode is 8-bit, because banding is then measured more accurately.

`vmaf.json` has the score of every frame under `frames`, and the mean, minimum, maximum and harmonic mean over the video under `pooled_metrics`, in the `vmaf` entry. Without `--json` the report is XML; `--csv` and `--sub` (subtitles) are the other formats. vmaf prints the mean score itself only when its error output is a terminal, as with `docker run -t`, so always write a report with `-o`.

Y4M is uncompressed, about 3 MB for each 1080p frame, or 4.5 GB for a minute of video at 24 frames per second. To score part of a video, add `-ss <start> -t <seconds>` before `-i` in both `ffmpeg` commands, or pass `--frame_cnt <n>` to vmaf.

Videos that are already Y4M or raw YUV need only vmaf, which `slim` also has. Raw YUV files carry no header, so give their size, chroma subsampling and bit depth:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/vmaf:slim \
  -r reference.yuv -d distorted.yuv --width 1920 --height 1080 --pixel_format 420 --bitdepth 8 -o vmaf.xml
```

## Inputs

The two videos must have the same size, chroma subsampling and bit depth, or vmaf stops with an error such as `dimensions do not match`. An encode at a lower resolution is scored at the resolution of its source, as upstream recommends: upscale it while decoding, for example with `-vf scale=1920:1080:flags=bicubic` in the second `ffmpeg` command.

vmaf pairs the frames in order and does not read frame rates or timestamps. Both videos need the same frame rate and must start on the same frame; `--frame_skip_ref <n>` and `--frame_skip_dist <n>` drop frames at the start of either one. When one video ends first, vmaf prints a warning, scores the frames it has paired and exits with status 0, so check that both have the same number of frames.

## Models and metrics

Without `--model`, vmaf uses `vmaf_v0.6.1`, the model for 1080p video watched from three times the screen height (3H). Every model that VMAF builds in is in the image and is loaded with `--model version=<name>`:

| Model | For |
|---|---|
| `vmaf_v0.6.1` | 1080p at 3H, the default |
| `vmaf_4k_v0.6.1` | 4K at 1.5H |
| `vmaf_v0.6.1neg`, `vmaf_4k_v0.6.1neg` | comparing encoders: no gain for sharpening or other enhancement |
| `vmaf_v1.0.16_3d0h` | VMAF v1, 1080p at 3H |
| `vmaf_v1.0.16_5d0h` | VMAF v1, 1080p on a phone (5H) |
| `vmaf_v1.0.16_1d5h_2160` | VMAF v1, 4K at 1.5H |
| `vmaf_v1.0.16_3d0h_2160` | VMAF v1, 4K at 3H, with scores up to 110 |
| `vmaf_v1.0.16_hfr_3d0h` and the other `_hfr` names | the VMAF v1 models for 50 and 60 frames per second |
| `vmaf_b_v0.6.3` | 1080p, with a confidence interval from bootstrapping |
| `vmaf_float_v0.6.1` and other `vmaf_float_` names | floating-point versions of the v0 models |

Upstream's [v1](https://github.com/Netflix/vmaf/blob/master/resource/doc/models_v1.md) and [v0](https://github.com/Netflix/vmaf/blob/master/resource/doc/models_v0.md) model pages explain the choice. Several models can be computed in one run when each gets a name, as in `--model version=vmaf_v0.6.1:name=v0 --model version=vmaf_v1.0.16_3d0h:name=v1`. A model file in JSON is loaded with `--model path=<file>`.

`--feature` adds other metrics to the report, one flag each: `psnr`, `psnr_hvs`, `float_ssim`, `float_ms_ssim`, `ciede` (CIEDE2000, a color difference) and `cambi` (banding in the distorted video). vmaf uses one thread unless `--threads <n>` is given.

There is no help option, and `-h` is short for `--height`. vmaf lists its options, and exits with status 1, when `-r` or `-d` is missing, so `--help` shows them after an error message. Run without arguments, the image prints the version.

## What is in the image

| Area | Details |
|---|---|
| Tool | `vmaf`, with libvmaf linked in statically |
| Input | Y4M and raw YUV, 4:2:0, 4:2:2 or 4:4:4, at 8, 10 or 12 bits, and 16 bits for raw YUV |
| Models | All built-in VMAF v0 and v1 models, including the floating-point ones |
| Metrics | VMAF, PSNR, PSNR-HVS, SSIM, MS-SSIM, CIEDE2000 and CAMBI |
| Output | XML, JSON, CSV and subtitles, per frame and pooled |
| CPU code | AVX2 and AVX-512 on x86-64, used when the CPU has them; NEON on arm64 |
| Default image | FFmpeg (`ffmpeg` and `ffprobe`), to decode, cut and scale the videos |

Not included: CUDA support, the libvmaf library and headers for building other software against it, the Python `vmaf` package with its training tools, and the model files, which vmaf does not need since the models are built in. FFmpeg in the default image is built without its `libvmaf` filter; use `vmaf` for the scores. The commit, the compiler and the build settings, such as the CPU code that was compiled, are in `/usr/local/share/randomcontainers/vmaf/buildinfo`.

## Default or slim

Use the default image (`latest`) to score encoded videos: it adds FFmpeg, which decodes, cuts and scales them into the Y4M files that vmaf reads (see [Quick start](#quick-start)). `slim` has vmaf alone. Use it when your videos are already Y4M or raw YUV, or to build your own image.

The default image includes FFmpeg, which is licensed under the GNU General Public License (GPL-3.0-or-later). Use `slim` if your policy excludes GPL. The default image is also published as `ghcr.io/randomcontainers/vmaf-ffmpeg`, built in the [vmaf-ffmpeg](https://github.com/randomcontainers/vmaf-ffmpeg) repository with the same contents and a different digest.

## Tags

`<version>` is a VMAF release such as `3.2.1`. `<minor>` and `<major>` are its shorter forms, `3.2` and `3`, and follow the newest release in that series.

| Default (with FFmpeg) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current VMAF version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation. The x86-64 images run on any x86-64 CPU and switch to AVX2 or AVX-512 code when the CPU supports it; `--cpumask` can turn those off.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no FFmpeg, so FFmpeg updates do not rebuild it. The packages vmaf needs are listed in `/usr/local/share/randomcontainers/vmaf/runtime-deps`. This image adds a model file of your own and scores every run with it:

```dockerfile
FROM ghcr.io/randomcontainers/vmaf:slim-ubuntu@sha256:...
COPY my_model.json /models/my_model.json
ENTRYPOINT ["tini", "--", "vmaf", "--model", "path=/models/my_model.json"]
```

The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The entrypoint of the slim image is `["tini", "--", "vmaf"]`. To pick up new VMAF releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/vmaf:latest \
  --repo randomcontainers/vmaf --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/vmaf-ffmpeg` are built in that repository, so verify them with `--repo randomcontainers/vmaf-ffmpeg` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/vmaf:latest --format '{{ json .SBOM }}'
```

VMAF publishes prebuilt binaries, which these images do not use, but no source tarball of its own, checksums or signed tags, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [Netflix/vmaf](https://github.com/Netflix/vmaf) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. The commit is also in `/usr/local/share/randomcontainers/vmaf/buildinfo`, and `vmaf --version` prints its start, as in `3.2.1-0-gf85a853`.

## Updates

The project checks the releases of [Netflix/vmaf](https://github.com/Netflix/vmaf/releases) every 15 minutes and skips pre-releases. A release is picked up once it is 24 hours old. The new version and the commit its tag points to are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new FFmpeg image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t vmaf:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

VMAF is released under the [BSD-2-Clause-Patent license](https://github.com/Netflix/vmaf/blob/master/LICENSE). `vmaf` also contains code that VMAF bundles from other projects: libsvm and the SSIM code from IQA (BSD-3-Clause), the PSNR-HVS code from Xiph, the Y4M reader from Daala and the CPU detection from dav1d (BSD-2-Clause), `x86inc.asm` from x264 (ISC), mkdirp and the CIEDE2000 code ported from av-metrics (MIT), and the pdjson JSON parser (Unlicense). The license texts are in `/usr/local/share/randomcontainers/vmaf/licenses/`, and links to the source are in `/usr/local/share/randomcontainers/vmaf/source`. pdjson's copy in VMAF carries no license text, so its `UNLICENSE` from [skeeto/pdjson](https://github.com/skeeto/pdjson) is kept in [licenses/](licenses/) and copied into the image.

The default image adds FFmpeg, licensed under GPL-3.0-or-later; see [randomcontainers/ffmpeg](https://github.com/randomcontainers/ffmpeg#licenses) for its sources and license files. libstdc++ and the other libraries from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
