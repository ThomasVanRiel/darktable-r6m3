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
gpg --recv-keys C4CBC150699956E2A3268EF5BB5CC8295B1779C9 \
               F10F9686652B0E949FCD94C318DCA123F949BD3B   # darktable signing keys, once
makepkg -si
```

It shows up in the app launcher as "darktable (R6 III)", next to the stock darktable if
both are installed.

`makepkg` keeps the downloads and the LibRaw clone in this directory (ignored by git), so a
rebuild works even if the upstream branch disappears, as long as this directory is kept.

## GPU AI (per machine)

The package bundles only the CPU ONNX Runtime, so the AI modules run on the CPU until
a GPU runtime is set up. For NVIDIA:

1. **CUDA and cuDNN.** You need CUDA 12 or 13 and cuDNN 9. Use `cuda` and `cudnn`
   from pacman (CUDA 13). CUDA 13 no longer supports Pascal (GTX 10xx) and Volta
   cards, so for those, install CUDA 12 and cuDNN 9 yourself, for example in your
   home directory, and add their `lib` directories to `LD_LIBRARY_PATH` when you start
   darktable.
2. **GPU ONNX Runtime.** Run darktable's installer from the unpacked source after a
   `makepkg`. It needs `curl` and `jq`, and installs to
   `~/.local/lib/onnxruntime-cuda/`:

   ```sh
   src/darktable-5.6.2/tools/ai/install-ort-gpu.sh --ep cuda13   # or cuda12
   ```

   It reads 5.6.2's `data/ort_gpu.json` by default. If the CUDA 12 download fails, pass
   darktable master's manifest with `--manifest <path>`.
3. **darktable.** In Preferences → AI, click *detect* (or browse to
   `libonnxruntime.so*`), restart, then download the models there. They go to
   `~/.local/share/darktable/models/`. Instead of the preference, you can also set
   `DT_ORT_LIBRARY=<path to libonnxruntime.so>` in a launcher script.

By default the CUDA runtime may use 75% of the GPU's memory. On a small GPU, lower that
with `DT_CUDA_MEM_LIMIT_MB=<MB>` in a launcher script, or with
`plugins/ai/cuda_mem_limit_mb` in `darktablerc`.

AMD (ROCm/MIGraphX) and Intel (OpenVINO) work the same way with `--ep migraphx`,
`rocm` or `openvino`; see `src/darktable-5.6.2/tools/ai/README.md`.

## Update

1. Edit `pkgver`, `_libraw_commit` or `_ort_ver`; reset `pkgrel=1` on a version
   bump, otherwise increment it.
2. `updpkgsums` (from `pacman-contrib`), then `makepkg -si`.
3. Commit, so every installed build has a matching commit.

When upstream LibRaw and darktable support the R6 III, drop the LibRaw
source, the `prepare()` lines that swap it in, and the patch. At that point the stock
`darktable` package should do the job, and this package can be removed.

## Check

```sh
/opt/darktable-r6m3/bin/darktable --version | grep -E '^darktable|LibRaw|OpenCL|AI '
pacman -Qi darktable-r6m3 | grep -E 'Version|Install Date'
```

The patch works if darktable names the camera `Canon` / `EOS R6 Mark III`,
the key of the bundled noise profile. A throwaway config dir keeps your library
out of it:

```sh
tmp=$(mktemp -d)
/opt/darktable-r6m3/bin/darktable-cli <some R6 III>.CR3 "$tmp/out.jpg" \
  --core --configdir "$tmp/config" -d sql 2>&1 | grep -m1 -o "INSERT  INTO main.cameras.*"
# expect: ... VALUES ('Canon', 'EOS R6 Mark III', 'EOS R6 Mark III')
rm -rf "$tmp"
```

To see the profile itself, open a CR3 in the darkroom and enable *denoise
(profiled)*: the profile list should offer `EOS R6 Mark III iso <n>` rather
than only "generic poissonian".

## Remove

```sh
sudo pacman -R darktable-r6m3
```
