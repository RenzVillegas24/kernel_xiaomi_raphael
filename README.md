# CrossPair kernel for the Redmi K20 Pro (`raphael`)

This is the CrossPair kernel fork for the Redmi K20 Pro / Mi 9T Pro
(`raphael`, Qualcomm SM8150, WCN3990). It is based on
[Evolution-X-Devices/kernel_xiaomi_raphael](https://github.com/Evolution-X-Devices/kernel_xiaomi_raphael)
and adds the monitor-mode transmit support required by the CrossPair userspace
stack.

This repository contains only the kernel side. The Android app, Rust helper,
Owly AWDL/AirDrop implementation, build scripts, and release workflows live in
[RenzVillegas24/CrossPair](https://github.com/RenzVillegas24/CrossPair).

## What this kernel adds

### Monitor-mode frame injection

The stock monitor interface is receive-only, and the WCN3990 firmware drops
management TX from monitor-type vdevs. This fork adds:

- A hidden, fully initialized STA adapter (`wlan_inj*`) created through the
  driver's normal adapter lifecycle.
- A monitor `ndo_start_xmit` hook that validates the radiotap header and frame
  bounds.
- A WMA helper that sends injected frames through the normal
  `wlan_mgmt_txrx_mgmt_frame_tx()` path, so the completion handler frees the
  nbuf and releases the mgmt-TX descriptor.
- Teardown, SSR, and cached-vdev cleanup so a stale injection vdev is never
  reused.

The hidden STA is used only as the WMI TX tag for injected frames. It is not
used as the AWDL peer identity.

### TX rate control

Injected data frames are sent at a fixed legacy rate instead of relying on the
firmware management-TX default:

- 54 Mbps OFDM, one spatial stream, 20 MHz
- retry limit 4
- injected frames capped at 1532 bytes

The 48 Mbps experiment was rejected; the current production branch keeps
54 Mbps.

### Owner-exit recovery

The kernel watches the userspace monitor owner through
`/sys/module/wlan/parameters/owly_owner_pid`. If the app exits, crashes, or is
killed, the driver returns to mission mode after a grace period so normal Wi-Fi
can recover instead of leaving the radio stuck in monitor mode.

The watchdog handles stale registrations, suspend/resume, failed reverts, and
mode-change races.

### Build fixes

The branch also carries GCC 13 build-warning fixes for the camera, IPA, WLAN,
audio, and compiler code touched by this configuration.

## How the pieces fit

```text
CrossPair app
  └─ foreground service + controller
       └─ owly-k20-helper (Rust)
            └─ Owly AWDL/AirDrop stack
                 └─ this kernel: monitor + hidden injection STA
```

Normal Wi-Fi, AP, and P2P operation are unchanged. The injection path is only
reachable when userspace explicitly opens monitor mode. The target radio is the
internal WCN3990; external USB Wi-Fi is out of scope.

## Compatibility

- Redmi K20 Pro / Mi 9T Pro only (`raphael`)
- Unlocked bootloader
- KernelSU root for the CrossPair app's radio-mode switch
- A boot image built from the matching kernel branch
- The CrossPair app and helper from the main repository

Do not flash a boot image built for another device or an unrelated kernel
branch. Keep a backup of the stock boot image before flashing.

Kernel building and boot-image packing are manual. The app repository's
`docs/SETUP.md` describes the requirements; this repository does not publish
prebuilt boot images or flashing automation.

## Branches

- `CrossPair/main` — production CrossPair kernel source
- `upstream-pr/airdrop-k20` — kernel changes submitted upstream to
  [Evolution-X-Devices/kernel_xiaomi_raphael](https://github.com/Evolution-X-Devices/kernel_xiaomi_raphael/pull/1)

Investigation, monitor-mode bring-up, and historical boot-image experiments
are kept outside the production branch.

## Validation

The production branch has been built with GCC 13, booted on a Redmi K20 Pro,
and exercised with AirDrop transfers to and from an iPhone, including
multi-file transfers, cancellation, app-exit recovery, and restoring normal
Wi-Fi. Observed throughput on the release configuration was approximately
2.1–2.6 MB/s TX and 3.6–3.9 MB/s RX.

## Privacy

The kernel assigns injection interface addresses at runtime. No MAC addresses,
BSSIDs, SSIDs, serial numbers, IP addresses, or tokens are hardcoded in this
repository.

## License

This is a Linux kernel tree and is distributed under the GNU General Public
License, version 2. See [`COPYING`](COPYING). Upstream copyright and license
notices are preserved.
