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
