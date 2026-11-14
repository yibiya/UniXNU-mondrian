# UniXNU-mondrian

UniXNU device support for the Redmi K60 (**mondrian**, Qualcomm **SM8475**).

- Firmware: [uefi-mondrian](https://github.com/Emilelu/uefi-mondrian)
- Core: [Emilelu/UniXNU](https://github.com/Emilelu/UniXNU) (branch `k60`)
- SoC: [Emilelu/UniXNU-sm8475-common](https://github.com/Emilelu/UniXNU-sm8475-common)

## Status (2026-11-14)

| Area | State | Notes |
| --- | --- | --- |
| UEFI boot chain | OK | ABL -> `fastboot boot` -> BootShim -> UEFI -> UniXNU menu |
| Display | OK | 1440x3200 panel, INTF1 measured at 60.5 Hz |
| Buttons | OK | KeypadDevice chain; `Pong keys 0/1/2 down/up` -> HID `sent=1` |
| Tools -> Advanced Logger | OK | 4 MB Advanced Logger + QR frames |
| XNU kernel | OK | builds on Linux (WSL) and macOS CI |
| Board ADT / payload / boot image | OK | 192 MiB payload + Android boot v4 image |
| UFS root (SHNGROOT) | OK | OS and SystemOS windows union-mounted on `/` |
| Data volume | OK | writable APFS Data window on the dedicated `unixnu` partition |
| Touch (Goodix BERLIN 9916R) | OK | `COORD-ON ack=80`, real validated `header=80` reports, kernel `type=1/2` contacts, BackBoard `HID dispatched type=1/2/3` |
| GPU (Adreno 730 v3) | OK, performance open | GMU/HFI, CP fences, Metal device and scanout all `status=0x0`; pacing still uneven (32 fps, 167/178 swaps one refresh late, 14.1 ms/frame in the driver) |
| Battery | BLOCKED | needs the real PAS1/exclusive GLINK handoff; no fabricated status |
| Audio / WiFi / Bluetooth / camera / sensors | TODO | research complete, implementation in progress |

### Plan

| Priority | Target | Criterion |
| --- | --- | --- |
| P0 | GPU frame pacing | Cut per-frame driver cost below the 16.7 ms budget so `came a refresh late` drops sharply |
| P1 | Touch multitouch and gestures | simultaneous contacts, edge back gesture, sustained cadence; confirm checksum rejections stop growing |
| P1 | Battery | real PAS1/exclusive GLINK handoff |
| P2 | Audio / WiFi / Bluetooth / camera / sensors | stage-by-stage implementation |


Measured: XNU reaches `launchd`, and `/private/var` mounts read-write from the
dedicated `unixnu` partition (`K60DATA mounted ro=0 fstype=apfs`,
`K60UFS write#N -> result=0`).

## Open blockers (2026-11-14)

1. **GPU frame pacing, not capability.** Every stage publishes `status=0x0`, but
   each frame costs ~14.1 ms in the driver against a 16.7 ms budget, so
   167 of 178 swaps arrive one refresh late. Work is bounded copy/completion
   batching, not another capability probe.
2. **Touch checksum rejections.** Two header and six contact checksum failures
   appeared in the same run that delivered real contacts. The per-report
   `fw_state`/`diff` diagnostics that could race the status clear are removed;
   the production image must show that the count stops growing.

## Build

Four sibling trees are required. Apply the core patch first:

```sh
git clone --recurse-submodules -b k60 https://github.com/Emilelu/UniXNU
git clone https://github.com/Emilelu/UniXNU-sm8475-common
git clone https://github.com/Emilelu/UniXNU-mondrian
git clone --recurse-submodules https://github.com/Emilelu/uefi-mondrian

cd UniXNU
../UniXNU-mondrian/scripts/k60-macos-preflight.sh      # core patches + auto-slide prelink wrapper
./unixnu defconfig mondrian_defconfig -D ../UniXNU-mondrian
./unixnu fetch -D ../UniXNU-mondrian
./unixnu build kernel -D ../UniXNU-mondrian
```

On Linux (WSL) the kernel and the prelink step both build without macOS; the
toolchain comes from `scripts/linux/setup-toolchain.py` and `unixnu fetch`
pulls `kernelcache.decompressed` from the IPSW. Only the Data seed
(`scripts/k60-make-data-seed.py`) still needs macOS.

```sh
python3 scripts/k60-prelink-local.py \
  --kernel mach.development.pong.unstripped \
  --prelinked out/ipsw/20G81/kernelcache.decompressed \
  --output kernel.macho
```

Firmware:

```sh
./unixnu build firmware --uefi ../uefi-mondrian -D ../UniXNU-mondrian
```

## Data safety

- Bring-up uses `fastboot boot` only: no partition is written.
- XNU root/Data use a dedicated `unixnu` partition (SHNGROOT OS/SystemOS windows
  plus a trailing Data window), never Android `userdata`; see
  `docs/DUAL-SYSTEM.md`.
- Never flash `persist` or run `flash all`.
- Run `--dry-run` and back up the manifest before any partition write.

## Known debt

- `boot-manifest-hash` is board-specific but lives in the shared OS profile of the
  core repo; it belongs in this repository's ADT section.
- Six build-environment patches sit at the tail of `patches/xnu/series` and have to
  be re-applied on every upstream bump.
- 22 further patches are not in `series` and have not been triaged.

## License

Device code is BSD-2-Clause-Patent; imported components keep their own SPDX
headers and licenses. See `THIRD_PARTY_NOTICES.md`.
