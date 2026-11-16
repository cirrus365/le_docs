---
title: Fixing “Unlock with biometrics” greyed out in the Bitwarden Firefox extension on macOS
description: A missing NativeMessagingHosts directory silently breaks Touch ID unlock in Bitwarden's Firefox extension on macOS. Diagnose and fix it in a few minutes.
weight: 11
categories: [tutorial]
tags: [bitwarden, firefox, macos, touch-id, troubleshooting]
---

Touch ID unlocks the Bitwarden desktop app and the Safari extension without complaint, but the Firefox extension shows **Unlock with biometrics** permanently greyed out. Reinstalling the extension, logging out and back in, and toggling browser integration all change nothing.

The cause is a missing native-messaging manifest. Firefox needs a small JSON file telling it how to hand the extension off to the desktop app, and on a number of macOS installs that file, and the directory meant to contain it, is never created. Without it the extension has no route to the desktop app, so it disables the option rather than offering a button that cannot work.

**Applies to:** Bitwarden desktop app and Firefox extension on macOS

## Symptoms

- The **Unlock with biometrics** checkbox in the extension's Account security settings is greyed out and cannot be ticked.
- Touch ID unlocks the desktop app itself without any problem.
- Biometric unlock works in Safari or a Chromium browser on the same machine.
- `~/Library/Application Support/Mozilla/NativeMessagingHosts/` does not exist, or exists but is empty.

## Before you start

- Bitwarden Desktop is installed from the Mac App Store. Bitwarden documents Touch ID as supported for that build rather than the downloads-page DMG.
- Touch ID is enabled in macOS system settings and unlocks the desktop app.
- **Allow browser integration** is ticked in the desktop app under Bitwarden → Settings → Security.
- You are on regular Firefox 87 or newer. Firefox ESR is explicitly unsupported for this feature.

## Method

1. **Confirm the directory is missing.** Run the check below. A `No such file or directory` result confirms the diagnosis; an empty directory produces the same symptom.

    ```bash
    ls -l ~/Library/Application\ Support/Mozilla/NativeMessagingHosts/
    ```

2. **Create the directory.** In most cases this alone is the fix, because the desktop app writes the manifest itself the next time it starts.

    ```bash
    mkdir -p ~/Library/Application\ Support/Mozilla/NativeMessagingHosts
    ```

3. **Restart in the correct order.** Quit Firefox completely with ⌘Q, quit Bitwarden Desktop, reopen Bitwarden Desktop and unlock it with Touch ID, wait a few seconds, then launch Firefox. The order matters: the extension probes for the desktop app at launch, so starting the browser first leaves the checkbox greyed out even with a valid manifest.

4. **Check whether the manifest appeared.** If `com.8bit.bitwarden.json` is now listed, continue to step 5. Otherwise, follow [the manual fix](#if-the-manifest-is-still-missing) first.

    ```bash
    ls -l ~/Library/Application\ Support/Mozilla/NativeMessagingHosts/
    ```

5. **Enable biometrics in the extension.** Open the extension, go to Settings → Account security, tick **Unlock with biometrics**, and approve the Touch ID prompt once to complete the pairing.

## If the manifest is still missing

### Write it manually

{{% callout tip %}}
Creating the empty `NativeMessagingHosts` directory is usually enough on its own, because Bitwarden populates the manifest on its next launch. Only write the JSON by hand if the folder stays empty.
{{% /callout %}}

Confirm where the app is installed first, because the `path` value has to match:

```bash
ls -d /Applications/Bitwarden.app
```

Then write the file:

```bash
cat > ~/Library/Application\ Support/Mozilla/NativeMessagingHosts/com.8bit.bitwarden.json <<'EOF'
{
  "name": "com.8bit.bitwarden",
  "description": "Bitwarden desktop <-> browser bridge",
  "path": "/Applications/Bitwarden.app/Contents/MacOS/desktop_proxy",
  "type": "stdio",
  "allowed_extensions": [
    "{446900e4-71c2-419f-a6a7-df9c091e268b}"
  ]
}
EOF
```

Verify both the manifest and the binary it points at:

```bash
cat ~/Library/Application\ Support/Mozilla/NativeMessagingHosts/com.8bit.bitwarden.json
ls -l /Applications/Bitwarden.app/Contents/MacOS/desktop_proxy
```

### Check the extension permission

In `about:addons` → Bitwarden → Permissions, confirm that **Exchange messages with programs other than Firefox** is allowed. If that prompt was dismissed during setup, biometrics cannot work regardless of the manifest.

## Notes

- If `desktop_proxy` is absent from the path in the manifest, the file points at nothing and the path needs correcting to wherever your build places it.

- Firefox forks such as Zen or Developer Edition may also need the manifest under that browser's own `NativeMessagingHosts` directory.

- A macOS username long enough to push `/Users/<username>/Library/...` past 104 characters will break the integration.

- If the checkbox is still greyed out after all of the above, the remaining suspect is an upstream regression where the native-messaging port connects but the IPC handshake with the desktop app times out after two seconds. There is no user-side workaround; Safari and Chromium browsers are unaffected until a patch ships.
