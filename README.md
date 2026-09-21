# cua-driver-musl-build

Builds [trycua/cua](https://github.com/trycua/cua) `cua-driver` for
**aarch64-unknown-linux-musl** (Alpine Linux / musl distros on ARM64) via GitHub
Actions. Upstream publishes no linux-arm64 asset, so this repo produces one.

## Contents

- `.github/workflows/build.yml` — builds the driver in a native Alpine 3.23 arm64
  container on an `ubuntu-24.04-arm` runner, then uploads the binary as an artifact.
- `skills/software-development/alpine-cua-driver-musl/SKILL.md` — Hermes skill
  documenting the full build/install/verify procedure.

## Build

Trigger from the Actions tab, or:

```sh
gh workflow run build.yml --ref main
```

The workflow checks out `trycua/cua` (input `ref`, default `main`) and runs the
entire build inside the container, so the host working directory never matters.
Output lands in the run's artifact `cua-driver-aarch64-unknown-linux-musl`.

## Install a release

Runtime libraries are required on Alpine:

```sh
apk add libx11 libxi libxtst libxrandr libxext
```

Then download and install the binary:

```sh
curl -fsSLO https://github.com/GegeDevs/cua-driver-musl-build/releases/latest/download/cua-driver-aarch64-unknown-linux-musl.tar.gz
curl -fsSLO https://github.com/GegeDevs/cua-driver-musl-build/releases/latest/download/SHA256SUMS
sha256sum -c SHA256SUMS
tar xzf cua-driver-aarch64-unknown-linux-musl.tar.gz -C /usr/local/bin
chmod +x /usr/local/bin/cua-driver
cua-driver --version
```

To replace a Hermes-managed install, install into the package release directory
and repoint `current` instead of writing to `/usr/local/bin`:

```sh
D=~/.cua-driver/packages/releases/0.28.2-aarch64-unknown-linux-musl
mkdir -p "$D"
cp cua-driver "$D/cua-driver"
ln -sfn "$D" ~/.cua-driver/packages/current
```

## Verify

```sh
file  ~/.local/bin/cua-driver   # should show /lib/ld-musl-aarch64.so.1
ldd   ~/.local/bin/cua-driver   # no "not found" lines
hermes computer-use doctor
```

The driver expects a live X11 display and an AT-SPI session bus. On a headless
host, see the `Headless X11` section of the skill.

## Notes

- The build links against **shared** X11 libraries (`-C target-feature=-crt-static`),
  because Alpine's X11 dev packages ship no static archives.
- The driver version in the release tag tracks the upstream `cua-driver` version;
  the `musl.N` suffix identifies this repo's build revision.
