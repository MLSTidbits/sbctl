# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Upstream and packaging

This is a downstream fork of [foxboron/sbctl](https://github.com/Foxboron/sbctl) maintained by Michael Schaecher. The primary purpose of this repo is to produce a Debian package (`debian/`) targeting Ubuntu Noble. Upstream changes are merged in; patches specific to this fork live in `debian/patches/`.

The package is built with `debhelper-compat 13`. The `debian/rules` file delegates the actual build to `make all` (man pages + binary). Build dependencies are `libpcsclite-dev`, `debhelper-compat (= 13)`, and `asciidoc-base`.

```bash
# Build the Debian package (from repo root)
dpkg-buildpackage -us -uc -b

# Or using debuild
debuild -us -uc -b
```

The `debian/changelog` tracks fork-specific releases (versioned as `sbctl (0.x) noble`). When bumping the package version, update `debian/changelog` following the existing format and run `dch` to add an entry.

## Commands

```bash
# Build
make build          # builds ./sbctl binary
go build -ldflags="-X github.com/foxboron/sbctl.Version=$(git describe)" -o sbctl ./cmd/sbctl

# Test
make test           # go test -v ./...
go test -v -run TestName ./...   # run a single test

# Lint
make lint           # go vet + staticcheck@v0.5.1

# Integration tests (requires QEMU + OVMF at /usr/share/edk2-ovmf/x64/OVMF_CODE.secboot.fd)
make integration

# Man pages (requires asciidoc/a2x)
make man
```

## Architecture

sbctl is a UEFI Secure Boot key manager. It runs as root and manipulates EFI variables and signs PE/COFF binaries.

### Package layout

**`github.com/foxboron/sbctl` (root package)** — core domain logic: signing files (`Sign`, `SignFile`), ESP detection (`GetESP`), bundle creation (`CreateBundle`, `GenerateBundle`), and the JSON file-signing database (`ReadFileDatabase`, `WriteFileDatabase`, `SigningEntries`).

**`cmd/sbctl/`** — Cobra CLI. Each subcommand lives in its own file (e.g. `sign.go`, `enroll-keys.go`). `main.go` wires the root command, builds a `config.State`, and passes it through `cobra.Command.Context()` via `stateDataKey{}`. Every subcommand retrieves state with `cmd.Context().Value(stateDataKey{})`.

**`config/`** — `Config` struct (YAML at `/etc/sbctl/sbctl.conf`, defaults to `/var/lib/sbctl`). `State` is the central runtime object passed everywhere — it holds the filesystem (`afero.Fs`), TPM opener func, Efivarfs handle, Yubikey reader, and resolved `Config`.

**`backend/`** — `KeyBackend` interface with three implementations: `FileBackend` (PEM private key), `TPMBackend` (TSS2 key via go-tpm), `YubikeyBackend` (PIV slot via go-piv). `KeyHierarchy` groups PK/KEK/Db backends and owns signing (`SignFile`) and verification (`VerifyFile`). Backend type is detected by inspecting PEM block type or JSON validity.

**`hierarchy/`** — Typed constants for the three UEFI key hierarchy levels (PK, KEK, Db) and their EFI variable mappings.

**`lsm/`** — Landlock LSM sandboxing. `LandlockRulesFromConfig` sets up path rules from config; `LandlockFromFileDatabase` adds per-file rules. Applied in `main.go` before command execution.

**`certs/`** — Built-in vendor certificates (Microsoft, etc.) bundled at compile time.

**`dmi/`** — DMI/SMBIOS reads for firmware quirk detection.

**`quirks/`** — Firmware quirk database; `fq0001.go` handles OptionROM detection via TPM eventlog.

**`fs/`** — Thin wrappers around `afero.Fs` for consistent file reads/writes across real and in-memory filesystems (used in tests).

### Key data flow

1. `main.go` opens TPM, reads/selects config, builds `config.State`, applies Landlock, stores state in Cobra context.
2. Subcommands retrieve `state` from context, call `backend.GetKeyHierarchy(state.Fs, state)` to load the active key backends.
3. Signing: `sbctl.Sign` → `backend.KeyHierarchy.SignFile` → `authenticode.PECOFFBinary.Sign` (from `go-uefi`).
4. EFI variable enrollment: subcommands use `state.Efivarfs` (from `go-uefi/efivarfs`) to write PK/KEK/Db EFI variables.

### Testing conventions

Unit tests use `afero.NewMemMapFs()` to avoid touching the real filesystem. Integration tests in `tests/integration_test.go` use `vmtest` to boot a QEMU VM with tianocore OVMF and exercise the full signing + enrollment flow; they require the `integration` build tag and a prepared OVMF image.
