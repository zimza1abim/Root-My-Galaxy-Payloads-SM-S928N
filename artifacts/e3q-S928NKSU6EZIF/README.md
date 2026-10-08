# Exploit artifact · SM-S928N / S928NKSU6EZIF

Device-specific CVE-2026-43499 app payload used by the [schema-v3 support
feed](../../support/targets-v3.json) entry `e3q-S928NKSU6EZIF-v1.2`.

## Artifact

- File: `cve-2026-43499-app.so`
- Size: **104,128 bytes**
- SHA-256:

```text
787762ec72f98a1c0fa9f1bda98abc0b97008cf261579d39ae35a7f623a9f5c1
```

This binary obtained root on the handset; the profile has four confirmed runs, the two most
recent on the first attempt. It is byte-identical to the build that ran on the device: the payload was
rebuilt from the target sources in `../../src/targets/e3q-S928NKSU6EZIF/` and compared by
digest.

The paired KernelSU daemon is described in the [artifact
reference](../../kernelsu/README.md). See the [port
report](../../docs/ports/e3q-S928NKSU6EZIF/port-report.md) for what was measured on this
firmware and what remains a candidate value.
