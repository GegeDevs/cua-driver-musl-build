---
name: alpine-cua-driver-musl
description: Use when fixing cua-driver on Alpine ARM64 musl.
version: 0.1.0
author: GegeDevs, Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [Alpine, ARM64, musl, cua-driver, computer-use]
    related_skills: []
---

# Alpine ARM64 cua-driver Skill

Build, install, and verify a musl-compatible `cua-driver` for Alpine ARM64 when the upstream installer provides an incompatible glibc binary or no Linux ARM64 asset.

## When to Use

- `hermes computer-use doctor` reports a missing or unstartable `cua-driver` on Alpine ARM64.
- The installed executable is an ELF glibc binary and the host only has the musl loader.
- A native ARM64 musl binary is needed for Hermes computer-use.

Don't use this for glibc Linux, macOS, Windows, or a real desktop session that already passes the doctor check.

## Prerequisites

- Alpine ARM64 host.
- GitHub CLI authenticated to a repository that can run Actions.
- A GitHub Actions ARM64 runner, preferably `ubuntu-24.04-arm`.
- X11 runtime libraries on Alpine: `libx11`, `libxi`, `libxtst`, `libxrandr`, and `libxext`.
- For headless operation: `xvfb`, `gsettings-desktop-schemas`, `dconf`, and AT-SPI support.

## Quick Reference

Use `terminal` for these checks:

    uname -m
    file /path/to/cua-driver
    cua-driver --version
    hermes computer-use doctor

Expected binary characteristics:

    ELF 64-bit LSB pie executable, ARM aarch64
    interpreter /lib/ld-musl-aarch64.so.1

## Procedure

1. Create or use a GitHub Actions repository with an ARM64 workflow. Checkpoint: the workflow uses `runs-on: ubuntu-24.04-arm`.
2. Check out `trycua/cua` and build inside one Alpine ARM64 container command. Keep the build, copy, and output directory inside that same command; do not run verification against a container-only path from the host.
3. Install build dependencies in the container:

       apk add --no-cache build-base cargo rustup pkgconf \
         dbus-dev libx11-dev libxtst-dev libxrandr-dev libxi-dev \
         glib-dev wayland-dev

4. Build the pinned driver from `libs/cua-driver/rust` using the toolchain required by the repository. Set `CC=gcc CXX=g++ AR=ar`. Alpine X11 packages provide shared libraries, so use `RUSTFLAGS="-C target-feature=-crt-static"` unless all dependencies are made static. Checkpoint: the output binary exists in the mounted artifact directory.
5. Verify the artifact with `file`, `sha256sum`, and `cua-driver --version`. Checkpoint: it reports the intended driver version and the musl loader.
6. Publish the binary, a tarball, and `SHA256SUMS` as GitHub Release assets. Use a versioned tag that identifies the driver version and musl build, for example `v0.28.2-musl.1`. Checkpoint: all three assets show `uploaded` in `gh release view`.
7. Install the binary into the driver package release directory and repoint the `current` symlink. Preserve any existing cursor theme or Wayland helper when present. Checkpoint: `/root/.local/bin/cua-driver --version` succeeds.
8. Install runtime libraries on Alpine:

       apk add libx11 libxi libxtst libxrandr libxext

   Checkpoint: `ldd cua-driver` reports no `not found` entries.
9. Run `hermes computer-use doctor` in the same environment that will run Hermes. For a headless host, provide a live X11 display and AT-SPI session bus. Checkpoint: `binary_version`, `platform_supported`, `ax_capability`, and `screen_capture_capability` are all green.

## Headless X11

Start a virtual display, then expose it to the shell that launches Hermes:

    Xvfb :99 -screen 0 1920x1080x24 -nolisten tcp
    mkdir -p /run/user/0
    dbus-daemon --session --address=unix:path=/run/user/0/bus --fork --nopidfile
    DISPLAY=:99 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/0/bus \
      /usr/libexec/at-spi-bus-launcher --launch-immediately

Set `DISPLAY=:99` and `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/0/bus` in the login environment only when the machine is intentionally headless. A real X11 desktop should use its own display and session bus.

## Pitfalls

- A symlink can make `which cua-driver` look valid while its target is absent or has the wrong ABI. Always inspect the target with `file`.
- `zsh` may cache a failed command lookup; run `hash -r` or start a fresh shell after replacing the binary.
- Do not use Ubuntu glibc X11 development libraries to link a musl target.
- Do not set `AR=musl-gcc`; use `AR=ar`.
- Do not assume `docker run` changed the host working directory. Put `cd`, build, and artifact copy inside the container shell and verify the mounted output from the host afterward.
- A passing binary check does not prove desktop access. Empty `DISPLAY`, missing X11 sockets, or missing `org.a11y.Bus` will keep doctor degraded.
- Xvfb provides a virtual desktop, not the user's physical desktop. Use a real X11 or XWayland session when controlling visible applications is required.

## Verification

Run all of the following:

    file /path/to/cua-driver
    ldd /path/to/cua-driver
    /path/to/cua-driver --version
    hermes computer-use doctor

The completed state is a musl ARM64 ELF, the expected driver version, no missing runtime libraries, and `hermes computer-use doctor` exiting successfully with X11, AT-SPI, and screen capture checks green.

For a release, verify:

    gh release view <tag> --json assets --jq '.assets[] | {name,size,state}'

Every expected release asset must have `state: uploaded`.
