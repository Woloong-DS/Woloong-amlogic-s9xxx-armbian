# Woloong Amlogic S9xxx Armbian

[中文说明](README.md) | [English](README.en.md) | Upstream: [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)

Builds **Woloong OS** Armbian images for Amlogic TV boxes (S905x3 / HK1-BOX and friends), forked from [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian).

- **Branding**: the login MOTD logo reads `Woloong OS`, and `VENDOR` / `VENDORPRETTYNAME` are `Woloong OS` as well — the official Armbian logo layout is kept, only the text changes.
- **Kernel**: taken from [Woloong-DS/Woloong-kernel](https://github.com/Woloong-DS/Woloong-kernel), version string like `6.18.55-Woloong`, with the drivers Klipper / CAN need.
- **Cadence**: images build on the 8th and 23rd of each month (20:00 UTC), half a month offset from kernel builds (1st / 16th), so each image build can pick up the freshest kernel.

## Customizations vs. upstream

| File | Kind | Purpose |
|------|------|---------|
| `.github/workflows/build-armbian-arm64-server-image.yml` | modified | Woloong defaults, 8th/23rd schedule, `\|\|` fallbacks on all inputs, `permissions: contents: write`, action points at this repository, overlay pre-check step |
| `.github/workflows/build-armbian-using-official-image.yml` | modified | `kernel_repo` defaults to `Woloong-DS/Woloong-kernel` |
| `.github/workflows/build-armbian-using-releases-files.yml` | modified | same |
| `.github/workflows/build-armbian-x86-server-image.yml` | modified | same |
| `.../common-files/etc/default/armbian-motd` | added | `VENDOR="Woloong OS"`, `MOTD_DISABLE="tips"` |
| `.../common-files/usr/local/bin/woloong-motd-fix` | added | idempotent brand repair (MOTD switch + both release files) |
| `.../common-files/etc/apt/apt.conf.d/99-woloong-branding` | added | `DPkg::Post-Invoke` runs the repair script after every apt/dpkg |
| `.../common-files/usr/sbin/armbian-fix` | modified | writes `Woloong OS` into `/etc/armbian-release` and `/etc/armbian-image-release` on first boot |
| `.../common-files/etc/custom_service/start_service.sh` | modified | calls the brand repair script at boot |
| `rebuild` | modified | rewrites `VENDOR`/`VENDORPRETTYNAME`, overlay copy logging and pre-check |

## How branding survives updates

`/etc/update-motd.d/10-armbian-header` sources `/etc/armbian-image-release`, `/etc/armbian-release` and `/etc/default/armbian-motd`, then prints `VENDORPRETTYNAME` (preferred) or `VENDOR` through `figlet -f small`. Three layers keep it in place:

1. **Image build**: `rebuild` rewrites `/etc/armbian-release`.
2. **First boot**: `armbian-fix` writes both release files — upstream writes `Armbian OS` here, this repository writes `Woloong OS`.
3. **Runtime self-heal**: `woloong-motd-fix`, invoked by the apt `Post-Invoke` hook and by `start_service.sh` at boot, restores the branding whenever something changes it back.

Layer 3 exists because `armbian-fix` reruns during first boot and kernel-update flows and would otherwise overwrite the image-build branding.

## Automation

Main workflow: `Build Armbian arm64 server image`

| Trigger | Detail |
|---------|--------|
| `schedule` | `0 20 8,23 * *` (UTC) = 04:00 Beijing time next day, twice a month |
| `workflow_dispatch` | manual run with inputs |
| `repository_dispatch` | external API trigger |

Inputs (defaults): `set_release=bookworm`, `armbian_board=s905x3-hk1`, `armbian_kernel=6.18.y`, `kernel_repo=Woloong-DS/Woloong-kernel`, `kernel_usage=stable`, `armbian_fstype=ext4`, `armbian_edition=server`, `builder_name=woloong`, `config_board=odroidn2`.

Steps: Checkout → **Verify Woloong overlay in action checkout** (fail fast if the branded `common-files` were not checked out with the action) → Initialize environment → Create virtual disk → Download Armbian source → Compile Armbian → Organize outputs → Upload → **Rebuild for target boards** (`uses: Woloong-DS/Woloong-amlogic-s9xxx-armbian@main`) → Upload.

## Artifacts

Release tag `Armbian_<release>_arm64_<edition>_<YYYY.MM>`, containing the ready-to-flash image (e.g. `Armbian_26.11.0_amlogic_s905x3-hk1_bookworm_6.18.55_server_2026.10.12.img.gz`) plus the intermediate rootfs/image.

## Flash and use

```bash
sudo armbian-install                                            # install to eMMC
sudo armbian-update -k 6.18.y -u stable -r Woloong-DS/Woloong-kernel   # upgrade kernel
```

## Verified on hardware

HK1-BOX (verified on real hardware): Armbian OS 26.11.0 / Debian 12 bookworm, `BOARD="S905x3-Hk1"`, kernel `6.18.55-Woloong`, MOTD logo rendered as `Woloong OS`, `VENDOR`/`VENDORPRETTYNAME` = `Woloong OS` in all three files, `gs_usb.ko` present, no failed systemd units.

## Staying in sync with upstream

```bash
git fetch upstream && git merge upstream/main
```

- Upstream hardcodes its own `/home/runner/work/_actions/ophub/...` path and uses a relative `common-files` path (which breaks after `cd build/output/images`), so workflows must `uses:` **this** repository rather than `ophub/amlogic-s9xxx-armbian`.
- Conflicts usually land in `rebuild`, `.github/workflows/*` and `common-files/usr/sbin/armbian-fix`; keep the Woloong branding changes.
- Push from a full clone: shallow/partial clones fail with `did not receive expected object ... remote unpack failed`.

## FAQ

- `Resource not accessible by integration` — GITHUB_TOKEN needs `permissions: contents: write` (already set) and the repo must allow read/write workflow permissions.
- `ln: failed to create symbolic link '/home/runner/work/_actions/ophub/...'` — upstream's hardcoded action path; resolved here by deriving the action directory from `${{ github.repository }}`.
- `cp: target directory '.../common-files/': No such file or directory` — upstream's relative path bug; fixed by pointing the workflow at this repository and gating it with the overlay pre-check.
- Repository changes do not show up in images until they are pushed to `main` (the workflow uses `@main`).

## Documentation & License

- Upstream English documentation (original `README.md`, preserved): [`README.ophub.md`](README.ophub.md)
- Upstream Chinese / Japanese documentation: [`README.cn.md`](README.cn.md), [`README.ja.md`](README.ja.md)
- Kernel compile/usage docs: [`compile-kernel/`](compile-kernel)
- License: see [`LICENSE`](LICENSE) (inherited from upstream)
