# Support feed

`targets-v3.json` is the schema-v3 payload feed for this profile. Add this repository as a
custom payload source in the Root My Galaxy app and pick the
**Galaxy S24 Ultra SM-S928N | EZIF v1.2** entry.

## Before first use: replace the URL placeholder

Both artifact URLs contain `zimza1abim/Root-My-Galaxy-Payloads-SM-S928N`. Replace `zimza1abim` with your GitHub
username after the first push:

```sh
sed -i '' 's|zimza1abim|your-github-username|g' support/targets-v3.json
```

Write artifact URLs in the **mutable branch form** (`.../zimza1abim/Root-My-Galaxy-Payloads-SM-S928N/main/<path>`). The
app pins them itself: it accepts a URL beginning with the source's own
`<repo>/<branch>/` prefix, strips it, and re-points the rest at the commit the source
resolved to. A URL that carries a commit hash in place of the branch matches no prefix and
is refused with *"The artifact repository is not allowed"*. Pin the **source** in the app
instead, not the URL here.

## Guard fields

`exactKernelRelease`, `exactBuildId`, `exactFingerprint`, `exactAndroidRelease`, and
`exactSdk` are matched exactly before anything is downloaded or run, and the native payload
re-checks them plus the security patch level. They are what keeps this profile from loading
on another firmware; do not relax them to widen compatibility.

Root was obtained on this handset four times, but that is not a stability claim — see the
[port report](../docs/ports/e3q-S928NKSU6EZIF/port-report.md).
