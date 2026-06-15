<p align="center">
  <img src="src/qt/res/icons/electron.png" alt="Electron" width="95">
</p>

# Electron Core 0.25.2

Electron Core 0.25.2 is the Electron-ELT port of Core v25.2. It keeps
Electron's chain identity, AuxPoW merged-mining rules, Blake-256 hash policy,
and wallet formats while adding the Taproot-era Core codebase, descriptor-wallet
support, SQLite wallet support, ZMQ, and Linux USDT tracepoints for hardened
release builds.

## Mainnet Consensus Changes In 0.25.2

Electron 0.25.2 follows the 0.15.21 SegWit activation and then activates the
next Core-compatible script rule sets in a staged order. Miners and pools
should use the daemon-provided block template version; do not manually rewrite
version bits or AuxPoW chain-id bits.

| Rule set | Mainnet policy in Electron 0.25.2 |
|---|---|
| SegWit (`BIP141` / `BIP143` / `BIP147`) | Already active from 0.15.21; buried at height `6197280`. No new SegWit signaling window in 0.25.2. |
| Subsidy ladder restore | Mainnet preserves the 0.15.21 flat `50 ELT` history before height `6175929`; from height `6175929`, 0.25.2 restores Electron's absolute-height legacy 0.8 ladder. Because `6175929` is already above the `1051200` ladder tier, the restored mainnet payout is `5 ELT` from activation onward. |
| `BIP34` coinbase height | Height activation at `6260635`; `BIP34Hash = uint256{}`. |
| `BIP65` / CLTV | Height activation at `6260635`; required for standard CLTV atomic-swap refunds. |
| `BIP66` / strict DER | Height activation at `6260635`. |
| Taproot (`BIP340` / `BIP341` / `BIP342`) | BIP9 deployment bit `2`, start `1782871200` (`2026-07-01 02:00:00 UTC`), timeout `1814407200` (`2027-07-01 02:00:00 UTC`), minimum activation height `6270715`. |

Only Taproot is a future BIP9-signaled deployment in 0.25.2. `BIP34`,
`BIP65`, `BIP66`, and buried SegWit are height rules. AuxPoW pools should use
`createauxblock` / `getauxblock` from Electron Core; the daemon computes the
correct top bits, Electron chain-id bits, and Taproot bit `2` during BIP9
`started` and `locked_in` states.

## About Electron

Electron is a Blake-256 AuxPoW coin in the BlakeStream family. It is merge-mined
with BlakeStream-family infrastructure and keeps its own network, address,
subsidy, and maturity rules.

- Uses the Blake-256 hashing algorithm, 8 rounds
- Based on the v25.2 Core codebase
- Uses the autotools build system (`./autogen.sh`, `./configure`, `make`)
- Supports legacy Berkeley DB wallets and descriptor SQLite wallets
- Keeps Electron txids on single SHA-256
- Uses HASH256/double SHA-256 for witness-v0 BIP143 signing
- Keeps BIP340/BIP341/BIP342 Taproot tagged hashes byte-compatible with the upstream BIP definitions

| Network Info | Value |
|---|---|
| Algorithm | Blake-256, 8 rounds |
| Block time | 1 minute |
| Difficulty retarget | Every 30 blocks |
| PoW limit | `0x000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffffff` |
| Coinbase maturity | 460 blocks |
| AuxPoW chain ID | `0x0007` |
| AuxPoW start height | `500000` mainnet, `0` testnet/regtest |
| Default P2P port | 6853 |
| RPC port | 6852 |
| Regtest RPC port | 18332 |
| Mainnet genesis | `00000063044a9b6cad9015b289de8a86a654d5a916fcaaeeefb5b9408448df40` |
| Mainnet Bech32 HRP | `elt` |
| Testnet Bech32 HRP | `telt` |
| Regtest Bech32 HRP | `relt` |

## Network Activation Notes

The SegWit activation block is height `6197280`. The STARTED period
`6197220-6197249` signaled SegWit bit `1` on `30/30` blocks, exceeding the
Electron `27/30` threshold. Post-active blocks no longer need to continue
setting SegWit bit `1`.

Regtest keeps Taproot always active by default for developer tests, and can use
`-vbparams=taproot:...` when a test needs to force a live BIP9 signaling window.
AuxPoW template tests should verify `createauxblock` / `getauxblock`, not only
standard `getblocktemplate`.

## Quick Start

```bash
git clone https://github.com/SidGrip/Electron-ELT.git
cd Electron-ELT
git checkout electron-25.2-auxpow-port
./build.sh --help
```

For most users, downloading a tested release artifact from GitHub Releases is
the simplest path. Use `build.sh` to build release artifacts locally.

## UPnP / miniupnpc Build Profiles

UPnP is only for desktop or home-router nodes that need automatic inbound P2P
port mapping. Nodes still sync normally through outbound peers without UPnP.

Pool, explorer, server, and Docker daemon builds should disable UPnP with
`--without-miniupnpc` so the binary has no `libminiupnpc.so.*` runtime
dependency.

UPnP-enabled Ubuntu builds need `libminiupnpc-dev` at build time and the matching
`libminiupnpc` runtime package on the target host.

## Upgrade Notes

Before starting Electron Core 0.25.2 on an existing data directory, close the
older wallet cleanly and back up any wallet files first.

When syncing 0.25.2 from old 0.8/0.15.21-era chains, header presync can look slow or restart because v25 verifies low-work header chains before storing them. For trusted bootstrap only, use `-minimumchainwork=0 -connect=<trusted-node>` and remove those options after the node catches up.

`peers.dat` is only the cached P2P address database. It is safe to remove or
rename when moving between major releases, and Electron will rebuild it on the
next start. If startup fails with `Invalid or corrupt peers.dat`, remove or
rename this file:

- Windows: `%APPDATA%\Electron\peers.dat`
- Linux: `~/.electron/peers.dat`
- macOS: `~/Library/Application Support/Electron/peers.dat`

If the block index or chainstate database cannot be reused after an upgrade,
restart once with `-reindex` to rebuild the local block database from the stored
block files:

```bash
electrond -reindex
```

Pruning is disabled by default (`-prune=0`), so a normal Electron Core node
keeps full block data. Public release nodes, explorers, pools, and bridge/watch
services should run unpruned unless they have a specific reason to discard old
block data.

For first-run testing of a new 0.25.2 build, use an isolated data directory so
the test does not touch an existing 0.15.21 wallet or chainstate:

```bash
electron-qt -datadir=/path/to/electron-25.2-test
```

## Build Options

```bash
./build.sh [PLATFORM] [TARGET] [OPTIONS]

Platforms:
  --native          Build natively on this machine (Linux, macOS, or Windows)
  --appimage        Build portable Linux AppImage (requires Docker)
  --windows         Cross-compile for Windows from Linux (requires Docker)
  --macos           Cross-compile for macOS from Linux (requires Docker)

Targets:
  --daemon          Build daemon only (electrond + electron-cli + electron-tx)
  --qt              Build Qt wallet only (electron-qt)
  --both            Build daemon and Qt wallet (default)

Docker options:
  --pull-docker     Pull prebuilt Docker images from Docker Hub
  --build-docker    Build Docker images locally from repo Dockerfiles
  --no-docker       For --native on Linux: build directly on the host

Other options:
  --hardened-release
                   Native Linux release profile: enable SQLite, ZMQ, and USDT
                   and fail the build if configure disables any of them
  --jobs N          Parallel make jobs
```


<!-- BEGIN electrum-build -->
### Electrum Wallet

Build the Electron ([Electrum](https://github.com/BlueDragon747/Blakestream-Electrum)) wallet by
choosing a target (linux/windows build in an **amd64** container, so any amd64 Docker host — Linux,
Windows, or an Intel Mac — can build either; only the macOS app needs a Mac):

```bash
./build-electrum.sh linux      # Linux AppImage    (amd64 Docker host)
./build-electrum.sh windows    # Windows .exe      (amd64 Docker host)
./build-electrum.sh macos      # macOS .dmg/.app   (on a Mac)
./build-electrum.sh all        # everything buildable on this host
```

Artifacts land in `outputs/Electrum/ELT/`, named `Electrum-ELT-<version>`.

The wallet builder is the shared multicoin repo
**[BlueDragon747/Blakestream-Electrum](https://github.com/BlueDragon747/Blakestream-Electrum)** — it also builds
all six BlakeStream wallets at once (`build-single-wallets.sh`) and the ElectrumX **server** Docker
image (`build-electrumx.sh`). `build-electrum.sh` auto-clones it when no local checkout is found.
<!-- END electrum-build -->

## Platform Build Instructions

### Native Linux

Direct host build:

```bash
./build.sh --native --both --no-docker
```

Docker build using the configured native image:

```bash
./build.sh --native --both --pull-docker
```

Recommended hardened Ubuntu 26 release build:

```bash
DOCKER_NATIVE=sidgrip/native-base:26.04 \
  ./build.sh --native --both --build-docker --hardened-release --jobs 5
```

Native Linux build outputs are bare Ubuntu-native binaries in
`outputs/Ubuntu-XX/`, where `XX` comes from the host or Docker image release.
The builder supports Ubuntu `20.04`, `22.04`, `24.04`, and `26.04`; current
release lanes use Ubuntu `22.04`, `24.04`, and `26.04`.

Each Ubuntu output folder includes `install-deps.sh`, `README.md`,
`build-info.txt`, `config.log`, `test-config.ini`, `electron.conf`, the daemon
tools, and the Qt wallet artifacts when `--qt` or `--both` is used. Berkeley DB
`4.8.30.NC` is bootstrapped into the repo cache by the builder rather than
installed from apt.

The hardened Linux release profile requires:

- `USE_BDB=true`
- `USE_SQLITE=true`
- `ENABLE_ZMQ=true`
- `ENABLE_USDT_TRACEPOINTS=true`

### AppImage

```bash
./build.sh --appimage --pull-docker
```

The AppImage build uses `sidgrip/appimage-base:22.04` and writes
`Electron-0.25.2-x86_64.AppImage`, `README.md`, and `build-info.txt` to
`outputs/AppImage/`. It is intended for Ubuntu `22.04+`. Direct launch requires
`libfuse2` on Ubuntu `22.04` and `libfuse2t64` on newer Ubuntu releases; the
fallback launch path is `--appimage-extract-and-run`.

### Windows

```bash
./build.sh --windows --both --pull-docker
```

Windows release artifacts come from the MXE cross-compile container. Native
Windows builds are diagnostic only and are not the release lane.

### macOS

Native macOS build:

```bash
./build.sh --native --both --no-docker
```

Cross-build from Linux:

```bash
./build.sh --macos --both --pull-docker
```

The default Linux-to-macOS cross-build uses the `sidgrip/osxcross-base:sdk-26.2`
container with the depends system and `CONFIG_SITE`. Native macOS builds use
Homebrew dependencies and write to `outputs/Macosx/`. USDT/eBPF tracing is
Linux-only and is not a macOS release requirement.

## Output Structure

```text
outputs/
├── AppImage/
│   ├── Electron-0.25.2-x86_64.AppImage
│   ├── README.md
│   └── build-info.txt
├── Macosx/
│   ├── Electron-Qt.app
│   ├── electron-cli-0.25.2
│   ├── electron-qt-0.25.2
│   ├── electron-tx-0.25.2
│   ├── electron-wallet-0.25.2
│   ├── electron-util-0.25.2
│   ├── electrond-0.25.2
│   ├── electron.conf
│   ├── build-info.txt
│   ├── config.log
│   └── test-config.ini
├── Ubuntu-XX/
│   ├── README.md
│   ├── install-deps.sh
│   ├── electron.conf
│   ├── electron.desktop
│   ├── electron-256.png
│   ├── electron-cli
│   ├── electron-qt
│   ├── electron-tx
│   ├── electron-wallet
│   ├── electron-util
│   ├── electrond
│   ├── build-info.txt
│   ├── config.log
│   └── test-config.ini
├── Windows/
│   ├── electron-cli-0.25.2.exe
│   ├── electron-qt-0.25.2.exe
│   ├── electron-tx-0.25.2.exe
│   ├── electron-wallet-0.25.2.exe
│   ├── electron-util-0.25.2.exe
│   ├── electrond-0.25.2.exe
│   └── build-info.txt
└── release/
    ├── Electron-0.25.2-x86_64.AppImage
    ├── Electron-0.25.2-Ubuntu-22-x86_64.tar.gz
    ├── Electron-0.25.2-Ubuntu-24-x86_64.tar.gz
    ├── Electron-0.25.2-Ubuntu-26-x86_64.tar.gz
    ├── Electron-0.25.2-macOS-x86_64.tar.gz
    ├── Electron-0.25.2-Windows-x86_64.zip
    └── SHA256SUMS
```

`Ubuntu-XX` is the detected host or Docker image release, such as `Ubuntu-20`,
`Ubuntu-22`, `Ubuntu-24`, or `Ubuntu-26`. Current release packaging produces
`Ubuntu-22`, `Ubuntu-24`, and `Ubuntu-26` archives. The `outputs/release/`
folder is created by release packaging flows and may be absent after a single
platform build.

## Docker Images

When using `--pull-docker`, the build script uses these prebuilt images:

| Image | Purpose |
|---|---|
| `sidgrip/native-base:20.04` | Native Linux Ubuntu 20.04 build |
| `sidgrip/native-base:22.04` | Native Linux Ubuntu 22.04 build |
| `sidgrip/native-base:24.04` | Native Linux Ubuntu 24.04 build, default native Docker image |
| `sidgrip/native-base:26.04` | Native Linux Ubuntu 26.04 build, hardened release lane |
| `sidgrip/appimage-base:22.04` | Ubuntu 22+ AppImage build |
| `sidgrip/mxe-base:latest` | Windows cross-compile |
| `sidgrip/osxcross-base:sdk-26.2` | macOS cross-compile |

## Multi-Coin Builder

For coordinated BlakeStream family wallet builds, release staging, and
installer workflows, see the Blakestream Installer repository:
https://github.com/SidGrip/Blakestream-Installer.

## License

Electron Core is released under the terms of the MIT license. See
[COPYING](COPYING) for more information or see
https://opensource.org/licenses/MIT.
