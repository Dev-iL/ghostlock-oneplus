# Recover kernel symbols from an Android boot image

This is a Linux-first, clean-slate workflow. It produces:

```text
analysis/kernel       extracted kernel payload
analysis/vmlinux      reconstructed ELF
analysis/kallsyms.txt compact readelf output sorted by address
```

The normal path does not build Magisk and does not install the Android NDK. The
kernel extractor is a small Python standard-library parser for ordinary AOSP
boot images. The Magisk checkout is still synchronized because it provides the
audited native boot-image implementation and an optional `magiskboot` fallback
for vendor-specific or otherwise unusual images.

`vmlinux-to-elf` itself has four runtime dependencies. They are declared in the
`reconstruct_vmlinux.py` `uv` script header, so `uv` selects a compatible Python
and creates/reuses its managed environment automatically. No global `pip`
install is needed.

## 1. Put the files together

For this checkout, the scripts are in:

```sh
export KALLSYMS_TOOLS_DIR="$HOME/repositories/ghostlock-oneplus/tools/kallsyms"
```

If these files came from a gist, set `KALLSYMS_TOOLS_DIR` to the directory that
contains `kallsyms.py`, `bootstrap.py`, `preflight.py`, `extract_kernel.py`,
`reconstruct_vmlinux.py`, `export_kallsyms.py`, optional `find_kallsyms.py`, and
`common.py`.

The scripts use these required repository variables:

```sh
export MAGISK_REPO_DIR="$HOME/repositories/Magisk"
export VMLINUX_TO_ELF_REPO_DIR="$HOME/repositories/vmlinux-to-elf"
```

Optional URL overrides are available for mirrors or local test repositories:

```sh
# export MAGISK_REPO_URL=https://github.com/topjohnwu/Magisk.git
# export VMLINUX_TO_ELF_REPO_URL=https://github.com/marin-m/vmlinux-to-elf.git
```

Do not point either variable at a directory containing unrelated files. The
bootstrapper never deletes or resets a directory: it clones a missing path,
fast-forwards a clean matching checkout, and stops if the checkout is dirty,
detached, diverged, or has the wrong `origin`.

## 2. Install the small host tool set

On Debian or Ubuntu:

```sh
sudo apt update
sudo apt install --yes curl git binutils
```

On Fedora:

```sh
sudo dnf install curl git binutils
```

On Arch:

```sh
sudo pacman -S --needed curl git binutils
```

Install `build-essential` (or the equivalent host C compiler package) if the
preflight reports that a compiler is missing or `uv` has to build a Python
dependency from source:

```sh
sudo apt install --yes build-essential
```

Install `uv` using the official installer, then start a new shell if the
installer asks you to refresh `PATH`:

```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
command -v uv
uv python install 3.12
```

The scripts request Python 3.9 or newer. `uv python install 3.12` is simply a
predictable choice; the script metadata remains the source of the compatibility
constraint.

## 3. Run the automated workflow

From a directory containing `boot.img`, this is the complete workflow:

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/kallsyms.py"
```

It reads `boot.img` and writes `kallsyms.txt`. To choose explicit paths and keep
the intermediate kernel and `vmlinux` files for inspection:

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/kallsyms.py" \
  /home/iliya/android/boot.img "$PWD/kallsyms.txt" \
  --work-dir "$PWD/analysis"
```

The main script performs these stages automatically and stops on the first
failure:

1. Clone or fast-forward both repositories and initialize Magisk's submodules.
2. Run the preflight checks.
3. Extract the kernel from the AOSP boot image.
4. Reconstruct `vmlinux` with the synchronized `vmlinux-to-elf` source.
5. Run `readelf --symbols --wide vmlinux`, project each symbol to
   `address TYPE name`, and atomically write `kallsyms.txt`.

The normal path does not build Magisk or install the Android NDK. Rerunning the
command updates only clean, matching, fast-forwardable checkouts and replaces
the output atomically. A dirty, detached, wrong-remote, or diverged repository
is reported instead of being changed destructively.

## Troubleshooting fallback: run stages separately

Use the individual commands below only when the main script identifies a
specific failing stage or when you need to inspect an intermediate artifact.
They are intentionally shown in dependency order.

### Bootstrap repositories

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/bootstrap.py"
```

This clones missing repositories, fast-forwards clean existing checkouts, and
updates Magisk's recursive submodules. It does not build either project.

### Preflight

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/preflight.py" \
  --boot-image "$PWD/boot.img"
```

The preflight is read-only with respect to the repositories. It checks Git,
`uv`, both repository identities and working trees, Magisk submodules and boot
sources, the `uv`-managed Python/dependency environment, all three upstream
CLI entry points, and `readelf` (falling back to `llvm-readelf`). It also
reports whether the optional `nm` fallback and a host C compiler are present.
A missing compiler is a warning on the normal path because a platform wheel
may be available; use the compiler if dependency resolution needs a source
build.

`MAGISKBOOT_PATH` is intentionally only a warning here. The standard extractor
does not need it.

### Extract the kernel

Make an output directory and run the standard-library extractor:

```sh
mkdir -p analysis
uv run --script "$KALLSYMS_TOOLS_DIR/extract_kernel.py" \
  boot.img analysis/kernel \
  --dtb-output analysis/kernel.dtb
```

For AOSP boot header versions 0 through 4, the script reads the kernel size
and page layout, copies the kernel section, and separates a valid appended DTB
when one is present. It leaves compression intact; `vmlinux-to-elf` can handle
the supported compressed/raw kernel formats.

For an image that needs Magisk's broader format handling, first build only the
host `magiskboot` target. This is the optional path that requires the Android
NDK and Rust toolchain:

```sh
export ANDROID_HOME="$HOME/Android/Sdk"
(cd "$MAGISK_REPO_DIR" && ./build.py ndk)
(cd "$MAGISK_REPO_DIR" && ./build.py native magiskboot)
```

Magisk's current build check expects its own ONDK layout at
`$ANDROID_HOME/ndk/magisk` and an `ONDK_VERSION` of `r30.1`. A stock NDK
directory—even a locally installed r29—does not satisfy that check when placed
there directly. Keep a stock NDK separate and let `./build.py ndk` install the
matching ONDK if this optional fallback is needed.

Choose the ABI directory containing the host executable and set it explicitly:

```sh
export MAGISKBOOT_PATH="$MAGISK_REPO_DIR/native/out/x86_64/magiskboot"
uv run --script "$KALLSYMS_TOOLS_DIR/preflight.py" \
  --require-magiskboot --check-native-build
uv run --script "$KALLSYMS_TOOLS_DIR/extract_kernel.py" \
  boot.img analysis/kernel --dtb-output analysis/kernel.dtb
```

`magiskboot unpack` also decompresses a compressed kernel. On Windows, point
`MAGISKBOOT_PATH` at the `.exe` build if you choose this optional route; using
WSL is usually simpler for building it.

### Reconstruct `vmlinux`

Run the wrapper through `uv` so its inline dependency metadata is honored:

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/reconstruct_vmlinux.py" \
  analysis/kernel analysis/vmlinux
```

The wrapper imports the synchronized source directly from
`VMLINUX_TO_ELF_REPO_DIR`, while `uv` supplies its declared dependencies. It
also verifies that the output is non-empty because the upstream tool can report
some architecture failures without leaving a useful output file.

If automatic architecture detection is wrong, pass the upstream overrides. For
example, AArch64 uses ELF machine value 183:

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/reconstruct_vmlinux.py" \
  analysis/kernel analysis/vmlinux \
  --e-machine 183 --bit-size 64
```

Useful validation commands, when installed, are:

```sh
file analysis/vmlinux
readelf -h analysis/vmlinux
```

The input must contain enough kernel metadata for reconstruction. A kernel
built without `CONFIG_KALLSYMS` may still produce an ELF, but it will not have
the symbol information needed for the final step.

### Export the symbol table

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/export_kallsyms.py" \
  analysis/vmlinux analysis/kallsyms.txt
```

The default output is a compact, address-sorted projection of:

```sh
readelf --symbols --wide analysis/vmlinux
```

The generated lines have three fields: `address TYPE name`, where `TYPE` is
readelf's ELF type such as `FUNC` or `OBJECT`. This is the format consumed by
`tools/extract_target.py` and avoids the misleading `T`/`t` classification that
`nm` can produce for data objects in a reconstructed ELF with a merged WAX
section. `readelf` is supplied by the `binutils` package already installed in
the Linux setup above; no additional library is required. Set `READELF_PATH`
when the executable is not on `PATH`.

To use the older `nm` convention as a troubleshooting fallback:

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/export_kallsyms.py" \
  analysis/vmlinux analysis/kallsyms-nm.txt --format nm
```

This is equivalent to `nm -n analysis/vmlinux`, with `-n` making the result
address-sorted. Set `NM_PATH` or pass `--nm /path/to/nm` when needed. Neither
export is exactly the `/proc/kallsyms` four-column format. If that format is
required, use the companion wrapper around the cloned project's embedded-table
finder:

```sh
uv run --script "$KALLSYMS_TOOLS_DIR/find_kallsyms.py" \
  analysis/kernel analysis/embedded-kallsyms.txt
```

That file is `/proc/kallsyms`-like and is obtained from the kernel's embedded
table. It is a different artifact from the readelf/nm vmlinux output; use
whichever format your consumer expects.

### Identify the kernel release

`extract_target.py --kernel-release` expects the exact release string returned
by `uname -r`. On a running device, fetch it with:

```sh
adb shell uname -r
```

For a quick check on an uncompressed `boot.img`, the same banner may be
visible directly:

```sh
strings -a boot.img \
  | sed -n 's/^Linux version \([^ ]*\).*/\1/p' \
  | head -n 1
```

For a `boot.img`, run the workflow above through the reconstruction step, then
extract the release from the reconstructed kernel banner:

```sh
strings -a analysis/vmlinux \
  | sed -n 's/^Linux version \([^ ]*\).*/\1/p' \
  | head -n 1
```

The kernel in a boot image may be compressed, so searching `boot.img` directly
is not reliable. The resulting release can be passed to the offset extractor:

```sh
kernel_release=$(strings -a analysis/vmlinux \
  | sed -n 's/^Linux version \([^ ]*\).*/\1/p' \
  | head -n 1)
python tools/extract_target.py \
  --kallsyms analysis/kallsyms.txt \
  --kernel-release "$kernel_release"
```

On PowerShell, the equivalent running-device command is:

```powershell
$kernelRelease = (adb shell uname -r).Trim()
python tools\extract_target.py `
  --kallsyms analysis\kallsyms.txt `
  --kernel-release $kernelRelease
```

For a reconstructed `vmlinux` on Windows, use `strings.exe` and pass the
release printed by the same banner search to `--kernel-release`.

## Windows automated path

The Python scripts and standard AOSP extractor are cross-platform. Install Git
for Windows, `uv`, and LLVM (for `llvm-readelf.exe`; `llvm-nm.exe` remains the
fallback). Visual Studio Build Tools are
only needed if dependency installation falls back to compiling a package. The
Magisk native fallback is not part of the baseline Windows path; use WSL if it
is needed.

In PowerShell:

```powershell
irm https://astral.sh/uv/install.ps1 | iex
uv python install 3.12

$env:KALLSYMS_TOOLS_DIR = "C:\work\kallsyms"
$env:MAGISK_REPO_DIR = "$HOME\src\Magisk"
$env:VMLINUX_TO_ELF_REPO_DIR = "$HOME\src\vmlinux-to-elf"

uv run --script "$env:KALLSYMS_TOOLS_DIR\kallsyms.py" `
  "$PWD\boot.img" "$PWD\kallsyms.txt" `
  --work-dir "$PWD\analysis"
```

The main script performs clone/update, preflight, extraction, reconstruction,
and export in one run. If LLVM is not on `PATH`, set `READELF_PATH` to
`C:\Program Files\LLVM\bin\llvm-readelf.exe` before invoking it. Git Bash can use
the Linux commands with the same scripts. Paths containing spaces are safe
because the scripts invoke subprocesses without a shell.

## Troubleshooting decisions

- `ANDROID!` not found: the image may be vendor-specific; set
  `MAGISKBOOT_PATH` and rerun extraction.
- `VNDRBOOT` detected: use the matching `boot.img`; `vendor_boot.img` normally
  carries vendor ramdisks and DTB data, not the generic kernel section.
- The reconstruction reports an unknown architecture: retry with
  `--e-machine` and `--bit-size`; use `--base-address` or `--file-offset` only
  when the kernel's layout requires them.
- `readelf` rejects the file: stop and inspect `file`/`readelf`; the
  reconstruction output is not a valid ELF yet. You can try the `nm` fallback
  with `--format nm`, but it uses section-derived one-letter types.
- The bootstrapper refuses to update a checkout: inspect it with `git status`
  and resolve the dirty, detached, wrong-remote, or non-fast-forward state
  yourself. The helper deliberately does not discard local work.
