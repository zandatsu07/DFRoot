> [!IMPORTANT]
> If you want to use your own ksud binary, you must compile from my fork: https://github.com/diabl0w/KernelSU

# DFRoot [DirtyFrag (CVE-2026-43284)]

The core of this code is fully credited to others. I merely combined ideas to make them all better 
and added some small improvements/features. 

Credits:
- Original PoC and various code: https://github.com/lsposed/lspromise
- Selinux Permissive kernel modules and various code: https://github.com/polygraphene/DFReroot
- Unprivileged XFRM socket method: https://github.com/combeng6th/DirtyInit

## Features

- Start on Boot
- Automatic soft reboot 
- RO Partition Protection
- Hide Selinux Modifications in KSU
- Shizuku not needed — regain root without WiFi!

> [!WARNING]
> I am not responsible for any damage to your device.

## Supported Devices

Ephemeral root for Samsung devices (and possibly others) w/ locked bootloaders vulnerable to DirtyFrag (CVE-2026-43284) 

| KMI Version | Verified |
|---|---|
| android12-5.10 | Yes |
| android13-5.10 | Untested |
| android13-5.15 | Yes |
| android14-5.15 | Untested |
| android14-6.1 | Not working - [accidental mitigation](https://github.com/V4bel/dirtyfrag/issues/23#issuecomment-4405314290) |
| android15-6.6 | Yes |
| android16-6.12 | Yes |
| android17-6.18 | Untested |

## How it works

The Android kernel decrypts AES-CBC ESP packets directly into the page cache of files open for `splice()`. By crafting `IV = AES_ECB_DEC(key, current_content) ⊕ desired_content`, any 16-byte-aligned block in a mapped shared library can be overwritten without write permission and without copy-on-write.

The exploit uses this primitive to patch shellcode into `libc++.so` in the kernel's page cache. The next privileged call to the hooked function runs the shellcode, which loads our custom kernel module via `insmod`. 

### Exploit chain

1. **IpSec transform** — App allocates a `UdpEncapsulationSocket` + SPI and builds an AES-CBC/HMAC-SHA256 ESP transform via `IpSecManager`.

2. **splicehelper → crash_dump64** — The splicehelper binary is spliced into `crash_dump64` via the CBC primitive. `crash_dump64` runs in the `crash_dump` SELinux domain (via exec label transition), which can open `vendor_file` labeled files (untrusted_app context cannot read these files so we need this bridge). The splicehelper serves two modes: splice mode (pipe a 16-byte page chunk out to the parent for write) and read mode (`argv[3]="r"`, write 16 bytes of file content to a pipe fd for IV computation).

3. **dirtyfrag.ko → vendor_file** — The kernel module is written via the crash_dump bridge (splicehelper splice mode) into a `vendor_file`-labeled file

4. **libc++ hook** (fires in init, uid=0, tid=1) — Shellcode is patched into `libc++.so` at `std::ostream::sentry::sentry()` (`_ZNSt3__113basic_ostreamIcNS_11char_traitsIcEEE6sentryC1ERS3_`). Triggered by: `createOrphanProcess()` double-forks so the grandchild is adopted by PID 1 (init); when init reaps the orphan its main thread (tid=1) calls through the hooked function. The shellcode:
   - Checks `getuid()==0` and `gettid()==1`; returns immediately otherwise
   - Creates `/dev/df` as a one-shot mutex (O_CREAT|O_EXCL) to prevent re-entry
   - Clones a worker child (parent returns to init immediately)
   - Worker forks a grandchild; grandchild writes `u:r:vendor_modprobe:s0` to `/proc/self/attr/exec` then execs `/vendor/bin/insmod <ko_target>`

5. **dirtyfrag.ko init** (runs as `vendor_modprobe`, uid=0) — The KO is loaded by `insmod` in the `vendor_modprobe` SELinux domain:
   - Writes `false` to `selinux_state` (global permissive)
   - Bypasses DEFEX via kprobes
   - Calls `call_usermodehelper` to run launch the `ksud` binary from our app's data dir
   - Module returns `-E2BIG` immediately after to self-unload

## Usage

Install KernelSU Manager (download & unzip manager file) from actions flow: 
https://github.com/tiann/KernelSU/actions/runs/35973514328

```sh
./build.sh
adb install -r dirtyfrag.apk
```

