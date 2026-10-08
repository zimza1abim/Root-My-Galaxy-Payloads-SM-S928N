# SM-S928N / S928NKSU6EZIF port report

**Port status: root was obtained on this handset four times. The two most recent runs, with
the payload published here, succeeded on attempt 1/24.** Device identity, kernel image,
structure layouts, and most symbols are confirmed from the exact firmware and the
recovered kernel. One runtime value remains a candidate rather than a measurement, and is
named below.

## Target identity

| Property | Result |
| :--- | :--- |
| Model / sales code | SM-S928N / `KOO` · bootloader stamp `KTC` · OMC group `OKR` |
| Firmware PDA / CSC | `S928NKSU6EZIF` / `S928NOKR6EZIF` |
| Fingerprint | `samsung/e3qksx/e3q:17/CP2A.260605.016/S928NKSU6EZIF:user/release-keys` |
| Android / SDK | 17 / 37 |
| Kernel release | `6.1.162-android14-11-34343818-abS928NKSU6EZIF` |
| Kernel image | Android boot image v4, ARM64, `_text = 0xffffffc008000000` |
| Security patch | 2026-09-05 |
| AP / BL / CP / CSC | all four archives match `S928NKSU6EZIF` |

Kernel config (`/proc/config.gz`, world-readable): `CONFIG_ARM64_VA_BITS=39`,
`CONFIG_ARM64_PA_BITS=48`, `CONFIG_RANDOMIZE_BASE=y`,
`CONFIG_RANDOMIZE_MODULE_REGION_FULL` not set.

## Values confirmed from this firmware

Symbol offsets for `ashmem_*`, `configfs_*`, `copy_splice_read`, `noop_llseek`,
`anon_pipe_buf_ops`, `ashmem_fops`, `kmalloc_caches`, `nfulnl_logger`, `init_task`,
`root_task_group`, `selinux_enforcing`, `sysctl_bootid`, and
`call_usermodehelper_exec_work` come from this kernel's own recovered ELF symbol table.

Two of them were cross-checked against an independently produced profile for the same
kernel release and matched exactly:

| Symbol | This profile | Independent profile |
| :--- | :--- | :--- |
| `init_task` | `0x0226f6c0` | `36107968` |
| `selinux_enforcing` | `0x02542618` | `39069208` |

Structure layout (`FAKE_WAITER_*`, `FAKE_TASK_*`, `FOPS_*`, `STRUCT_PAGE_*`) is identical
to the device-validated `e3q-S928BXXS6DZF2` profile — same device generation, same
`android14-6.1` KMI — and matches this kernel's BTF: `pi_waiters 0x938`,
`pi_tree_entry 0x18`, `pi_lock 0x924`, `pi_blocked_on 0x950`.

The vendor DTB carries `reserved-memory/gunyah_hyp_region@80000000` with
`reg = <0 0x80000000 0 0x00e00000>`.

## P0_PHYS_OFFSET: measured, and previously wrong

`P0_PHYS_OFFSET = 0x816e0000`.

The payload computes `__pa()` as `(linear_va - P0_PAGE_OFFSET) + P0_PHYS_OFFSET` and
`p0_data_alias()` inverts it, so this constant must be the kernel's `memstart_addr`. Read
directly, without root:

```
$ cat /proc/zoneinfo
Node 0, zone   Normal
        spanned  11004192
        present  2923738
  start_pfn:           530144      # 530144 * 4096 = 0x816e0000
Node 0, zone    DMA32
        spanned  0
        present  0
```

`start_pfn` was identical before and after a reboot, so the value is stable despite
`CONFIG_RANDOMIZE_BASE=y`. `present` (11.15 GiB) agrees with `MemTotal` (10.8 GiB), and
`spanned` (42 GiB) reflects a sparse layout — the device's reserved-memory map shows
regions at `0x880B00000` as well as the low window.

The earlier value, `0x80000000`, was taken from the DTB's gunyah anchor. That region is a
reserved hypervisor carveout below the first RAM page, so every computed alias landed
23 MiB away from its intended page. Note also that this kernel has **no DMA32 zone**,
which makes any DMA32-range special-casing dead weight here.

## Values that remain unverified

- `P0_KERNEL_PHYS_LOAD = 0xa8000000` — a same-family candidate. This BL archive contains
  ABL/UEFI, not `sboot.bin`. The LinuxLoader PE does yield the selector
  (`min EFI RAM descriptor base + 0x80000`, ARM64 `text_offset = 0`, region `0x05600000`),
  but the runtime descriptor base is not readable from an unprivileged shell, so the
  formula cannot be closed offline. Two constraints narrow it: `0x80080000` is below the
  first RAM page and inside the gunyah carveout, and the reserved-memory map leaves no
  86 MiB gap directly above the first RAM page, while `0xa8000000` falls in the free span
  between `0x9df00000` and `0xff800000`.
- `SKB_DATA_DELTA = -0x1000` / `SLIDE_S928_SKB_DATA_DELTA = -0xe80` are inherited from the
  `e3q-S928BXXS6DZF2` profile and were not re-measured against this build's live skb
  allocation.
- `SLIDE_PSELECT_WORD_SHIFT = 3` is a static result from the recovered ELF
  (`futex_wait_requeue_pi` waiter at `sp+0x98`, `core_sys_select` fd-set at `sp+0x50`,
  difference `0x18` = 3 qwords).

## Profile shape: what this target does not use

Earlier iterations of this port carried a large amount of target-specific machinery that
no device-validated profile uses — a fifth bank slot acting as a "production" FOPS slot, a
controlled `mm_struct` group reclaim, a separate fixed-physical data-alias verification
pass, a fresh-P0-session guard, and extra pselect state guards. That configuration never
once landed the `misc_fops` write, produced an unstable P0 gate (`result=1`, `-1`, `0` on
three consecutive runs), and panicked intermittently in
`rt_mutex_adjust_prio_chain` during the controlled reclaim.

The published profile turns all of it off and runs the shared upstream route: legacy
reclaim, four bank slots, tracefs slide with canonical FOPS addressing. The flags that
matter:

```c
APP_P0_RETRY_AFTER_WRITER        1   // without this the profile cannot succeed
APP_TRACEFS_KASLR_DIRECT         1   // also selects canonical FOPS addressing
APP_CONTROLLED_MM_GROUP_RECLAIM  0
APP_REQUIRE_FRESH_P0_SESSION     0
APP_FOPS_DATA_ALIAS_DIAG_ONLY    0
APP_FOPS_DATA_ALIAS_GATE_VERIFY  0
APP_FOPS_REUSE_VERIFIED_PAGE     0
```

`APP_TRACEFS_KASLR_DIRECT` is not only a slide-source switch: `data_addr_canonical` is set
from `source == "tracefs"`, and `data_addr()` then sends the FOPS write to the canonical
kernel-image address rather than the physical alias. The canonical write is the one that
rooted this device, so the alias variant should not be re-enabled without evidence.

The last three flags are interdependent. With the alias diagnostic off but
`APP_FOPS_REUSE_VERIFIED_PAGE` left on, nothing prepares the payload page and every
attempt dies immediately with `page=0` — fourteen consecutive attempts failed that way
before the combination was corrected.

## v1.1 / v1.2 changes

`APP_PAYLOAD_ATTEMPT_DELAYS_USEC` was a single element, so `attempt_delay_usec()` handed
every one of the 24 attempts the same 50 ms. It is now a 12-value sweep (50, 45, 55, 40, 60,
35, 65, 30, 70, 25, 75, 20 ms), and `APP_FOPS_ROUTE_COARSE_DELAY_USEC` was dropped in favour
of `APP_FOPS_ROUTE_USE_PSELECT_DELAY` so the route actually reads the supervisor's
per-attempt value instead of overriding it in the first branch of the chain. Note that the
`static delays[]` fallback inside `slide_app.c` cannot do this job: each attempt is its own
process, so `delay_index` resets and every attempt would take `delays[0]`.

`APP_FOPS_RESTORE_SLOT 1` adds a second crafted bank entry whose baked-in value is the
original `ashmem_fops`, with the same `misc_fops` target. `main.c` fires it when
`triggered && !verified`, which is the case that used to leave a dangling `misc_fops` behind.
It uses the `rb_erase` write rather than `configfs_write_once()` because the fake table is
precisely what cannot be trusted in that state. **Not yet exercised** — both runs since have
succeeded on the first attempt.

## Runtime evidence

```
[+] pipe physrw pid=2186  done=1 root=1 kaslr=1 read_ok=1 write_ok=1 rw64=1/1 uid=2000->0
[+] exploit completed attempt=5/24
[+] pipe physrw pid=19606 done=1 root=1 kaslr=1 read_ok=1 write_ok=1 rw64=1/1 uid=2000->0
[+] exploit completed attempt=4/24
```

Both runs then completed KernelSU late-load, control-channel verification, and
activation. SELinux remained enforcing throughout.

## Artifact

- `artifacts/e3q-S928NKSU6EZIF/cve-2026-43499-app.so`
- Size: **104,128 bytes**
- SHA-256: `787762ec72f98a1c0fa9f1bda98abc0b97008cf261579d39ae35a7f623a9f5c1`

The checked-in binary is byte-identical to the one that rooted the handset; the build was
reproduced from the published target sources and compared by digest.

## KernelSU pairing

`kernelsu/ksud-e3q-S928NKSU6EZIF-kdp`, KernelSU **3.2.5** (version code `32525`), built
with the Samsung KDP/RKP/DEFEX patch and this kernel's `vermagic`. Pair it with the 3.2.5
manager APK; a newer manager reports a driver version mismatch because #3613/#3628 changed
manager signing-block parsing.

`tracepoint_probe_register_prio`, which KernelSU 3.3.0 additionally imports, is present in
this kernel at `ffffffc0081de9fc` as a global text symbol, so a 3.3.0 pair is not blocked
on symbol availability — it simply has not been built for this kernel.

## Not done

No firmware, partition, CSC, or factory-reset operation was performed by the payload. The
exploit was not published to any upstream feed. A 3.3.0 KernelSU pair, a re-measurement of
`SKB_DATA_DELTA`, and a runtime confirmation of `P0_KERNEL_PHYS_LOAD` remain open.
