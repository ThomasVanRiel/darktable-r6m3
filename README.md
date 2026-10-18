# darktable-r6m3

Private pacman package: darktable with experimental Canon EOS R6 Mark III
support. It installs side by side with the stock `darktable` package, under
`/opt/darktable-r6m3`.

## What is pinned

| Part | Pin | Where |
|---|---|---|
| darktable | `release-5.6.2` tarball, sha256 + PGP signature | `source[0]` |
| LibRaw | MStraeten `r6mIII` branch at commit `aeb371cc` | `_libraw_commit` |
| camera table patch | `r6iii-modelmap.patch`, sha256 (same change as unmerged darktable PR #21433) | `source[4]` |
| CPU ONNX Runtime (bundled) | 1.24.4, sha256 matches GitHub's asset digest | `_ort_ver` |
| build flags | Arch's `extra/darktable` 5.6.2 flags, plus `USE_HEIF=Off` and the `/opt` prefix | `build()` |

Not pinned, and set up separately on each machine: the GPU ONNX Runtime and the CUDA
libraries (see below), and the AI models, which are downloaded through the GUI into
`~/.local/share/darktable/models/`.

Without the patch, darktable has no clean camera name for the R6 III: it shows
"camera is not fully supported", and the bundled R6 III noise profile never
matches.

## Install on a machine

```sh
git clone https://github.com/ThomasVanRiel/darktable-r6m3.git && cd darktable-r6m3
gpg --recv-keys C4CBC150699956E2A3268EF5BB5CC8295B1779C9   # darktable release key, once
makepkg -si
```

`makepkg` keeps the downloads and the LibRaw clone in this directory (ignored by git), so a
rebuild works even if the upstream branch disappears, as long as this directory is kept.

## Update

1. Edit `pkgver`, `_libraw_commit` or `_ort_ver`; reset `pkgrel=1` on a version
   bump, otherwise increment it.
2. `updpkgsums` (from `pacman-contrib`), then `makepkg -si`.
3. Commit, so every installed build has a matching commit.

When upstream LibRaw and darktable support the R6 III, drop the LibRaw
source, the `prepare()` lines that swap it in, and the patch. At that point the stock
`darktable` package should do the job, and this package can be removed.

## Per machine

### Workstation (RTX 4070 Ti Super)

One-time migration from the hand-built install, which lives at the same
prefix (pacman refuses to overwrite files it doesn't own):

```sh
cp ~/.config/darktable/library.db ~/.config/darktable/library.db.pre-r6m3-5.6.2   # see warning
sudo rm -rf /opt/darktable-r6m3
makepkg -si
```

The existing `/usr/local/bin/darktable` symlink and the
`/usr/local/share/applications/org.darktable.darktable.desktop` symlink both
point into `/opt/darktable-r6m3`, so they keep working unchanged.

GPU AI: CUDA and cuDNN come from pacman (`cuda`, `cudnn`), and the GPU ONNX Runtime is
in `~/.local/lib/onnxruntime-cuda/` (installed with
`tools/ai/install-ort-gpu.sh --ep cuda13`). The config is the shared
`~/.config/darktable/`, and Preferences -> AI is already set up there.

**Downgrade warning.** The previous build here was darktable master (5.7-dev).
The library schema is the same as 5.6.2's, but 717 images have
`highlights` v5 / `demosaic` v7 history, which 5.6.2 replaces with default
settings (`src/develop/develop.c:2632` in 5.6.2). 31 images use
`contrastntexture` and 2 use `satcurvergb`, which don't exist in 5.6.2. The
first edit of such an image in 5.6.2 rewrites its history and XMP.

### Babbage (GTX 1050, Pascal)

The package itself is identical. The GPU side stays as set up in
`SETUP-GUIDE.md`: CUDA 12.9 and cuDNN 9.10.2 in user space
(`install-cuda12-pascal-libs.sh`), ONNX Runtime from
`install-ort-gpu.sh --ep cuda12 --manifest <master ort_gpu.json>`, and the
`~/bin/darktable-r6iii` launcher. That launcher sets `LD_LIBRARY_PATH`, sets
`DT_CUDA_MEM_LIMIT_MB=2048`, and uses its own config dir.

Migration: install this package, change `PREFIX=` in `~/bin/darktable-r6iii`
to `/opt/darktable-r6m3`, check that it starts, then
`sudo rm -rf /opt/darktable-r6iii`. Both builds are 5.6.2, so the
`~/.config/darktable-r6iii` library needs no migration.

## Check

```sh
/opt/darktable-r6m3/bin/darktable --version | grep -E '^darktable|LibRaw|OpenCL|AI '
pacman -Qi darktable-r6m3 | grep -E 'Version|Install Date'
```

Opening an R6 III CR3 with `-d control` should log
`[noiseprofile] looking for maker 'Canon', model 'EOS R6 Mark III'` and find
the bundled profile.

## Remove

```sh
sudo pacman -R darktable-r6m3
```

On the workstation, also remove the `/usr/local` symlinks, or the
`darktable` command and the launcher will point at nothing.
