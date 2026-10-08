# KernelSU pair · SM-S928N / S928NKSU6EZIF

Targets `6.1.162-android14-11-34343818-abS928NKSU6EZIF`, KMI `android14-6.1`.

| Flavour | Version | Version code | Paired daemon |
| :--- | :--- | :--- | :--- |
| KernelSU | 3.2.5 | `32525` | `ksud-e3q-S928NKSU6EZIF-kdp` |

- Size: **4,882,688 bytes**
- SHA-256: `248725370336cd1557503a6ba525e34969b9b922bedc63f43290efc349d7b874`

The daemon embeds the module built for this exact kernel release with the Samsung
KDP/RKP/DEFEX patch and this kernel's `vermagic`. Late-load, control-channel verification,
and activation were confirmed on the handset in both successful runs, with SELinux
enforcing.

## Manager pairing

Install the **3.2.5** manager APK, `KernelSU_v3.2.5_32525-release.apk` from the upstream
`v3.2.5` release. The module verifies the manager APK's signing block, and KernelSU
#3613/#3628 changed how that block is parsed, so module and manager are a pair. A newer
manager — for example 3.3.0's `32601` — against this module reports a driver version
mismatch.

## 3.3.0

No 3.3.0 pair has been built for this kernel. The symbol 3.3.0 additionally imports,
`tracepoint_probe_register_prio`, is present in this kernel's recovered symbol table at
`ffffffc0081de9fc` as a global text symbol, so the upgrade is not blocked on symbol
availability.
