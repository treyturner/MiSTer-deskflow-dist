# MiSTer Deskflow

Deskflow lets you use the keyboard and mouse from one computer to control another over the network. This fork packages Deskflow for MiSTer FPGA so you can control retro computer cores over the network using the keyboard and mouse from your regular-use system without the hassle of switching inputs on a physical switch.

This is a **client-only** port. It requires you set up and run a Deskflow **server** on a Windows, macOS, or Linux machine connected to the keyboard and mouse you wish to use.

Builds are currently **preview-only** and require a subscription to [patreon.treyturner.info](https://patreon.treyturner.info).

## Why fork?

### Architecture & Dependencies

Upstream Deskflow is built for desktop operating systems. MiSTer is a much leaner, headless ARM Linux environment. Getting Deskflow running therefore requires a number of platform-specific changes.

- As MiSTer doesn't provide an X11 or Wayland desktop session, this port implements a new backend.
- Input has to be injected as synthetic keyboard and mouse devices so the active MiSTer core can see it.
- The stock MiSTer Linux image is older than the runtime expected by a modern Deskflow build, so this package ships the needed runtime libraries, OpenSSL modules, and XKB data alongside `deskflow-core`.
- Configuration is script-driven as the GUI can't be built without a real windowing system.
- Deskflow targets 64-bit architecture. Some bugs around this required fixing, there may be more, and I'm not interested in lobbying that this use-case is worth the project reopening 32-bit support.

### MiSTer-specific features

This fork implements features specific to MiSTer FPGA that don't have any place upstream:

- Dynamic desktop geometry
  - Mouse movement uses a core-specific MiSTer geometry lookup table and changes desktop resolutions on the fly based on the active core
- Cursor de-sync [mitigations](#mitigations)
  - Including a variety of desktop and mouse movement scaling options for tuning to taste
- Custom [clipboard sharing mechanics](#clipboard-sharing)
  - Text can be "pasted" from other screens into the MiSTer screen which simulates the needed keypresses with an optional delay between keys.
  - The most recently written MiSTer screenshot can be pushed to the Deskflow clipboard for convenient pasting on other screens.

## Pre-requisite: Deskflow server

The MiSTer client connects to a server that runs on the computer with your keyboard and mouse. See [`SERVER_SETUP.md`](SERVER_SETUP.md) for instructions.

## MiSTer client installation

1. Download `deskflow` into `/media/fat/Scripts` and make it executable by pasting this block directly into a shell:

   ```sh
   (
     if [ ! -d /media/fat/Scripts ]; then
       echo "/media/fat/Scripts not found" >&2
       exit 1
     fi

     url='https://raw.githubusercontent.com/treyturner/MiSTer-deskflow-dist/main/deskflow'
     dest='/media/fat/Scripts/deskflow'

     if curl -fsL -o "$dest" "$url" 2>/dev/null; then
       echo "Downloaded with system CA store."
     elif [ -f /etc/ssl/certs/cacert.pem ] && curl --cacert /etc/ssl/certs/cacert.pem -fsSL -o "$dest" "$url"; then
       echo "Downloaded using /etc/ssl/certs/cacert.pem."
     else
       echo "Missing CA bundle. Refresh MiSTer certs and try again. One-liner:" >&2
       echo '  (cd /etc/ssl/certs \' >&2
       echo '   && cp cacert.pem cacert.pem.bak \' >&2
       echo '   && wget --no-check-certificate https://curl.se/ca/cacert.pem -O cacert.pem)' >&2
       exit 1
     fi

     chmod +x "$dest"
     echo "Entrypoint downloaded and made executable: $dest ($(stat -c %s "$dest") bytes)"
   )
   ```

   <details><summary>Click for the commented version if you have any concerns...</summary>

   ```sh
   (
     # Fail fast if Scripts dir isn't found
     if [ ! -d /media/fat/Scripts ]; then
       echo "/media/fat/Scripts not found" >&2
       exit 1
     fi

     # Define for reuse
     url='https://raw.githubusercontent.com/treyturner/MiSTer-deskflow-dist/main/deskflow'
     dest='/media/fat/Scripts/deskflow'

     # MiSTer doesn't ship valid CA certs, but ignoring cert errors is risky.
     # Try fetching with curl as configured, however it's likely to fail SSL
     # verification if the CA bundle on this system hasn't yet been updated.
     if curl -fsL -o "$dest" "$url" 2>/dev/null; then
       echo "Downloaded with system CA store."
     # Next look for the cert bundle where it's supposed to be, and if it
     # exists, explicitly select it for use in the call.
     elif [ -f /etc/ssl/certs/cacert.pem ] \
     && curl --cacert /etc/ssl/certs/cacert.pem -fsSL -o "$dest" "$url"; then
       echo "Downloaded using /etc/ssl/certs/cacert.pem."
     else
       # Previous attempts failed, so it's now in the user's hands to
       # update the system's CA cert bundle to facilitate secure calls.
       echo "Missing CA bundle. Refresh MiSTer certs and try again. One-liner:" >&2
       echo '  (cd /etc/ssl/certs \' >&2
       echo '   && cp cacert.pem cacert.pem.bak \' >&2
       echo '   && wget --no-check-certificate https://curl.se/ca/cacert.pem -O cacert.pem)' >&2
       exit 1
     fi

     # A download succeeded; make the script executable
     chmod +x "$dest"
     # Log success as the script path and downloaded bytes
     echo "Entrypoint downloaded and made executable: $dest ($(stat -c %s "$dest") bytes)"
   )
   ```

   </details><br />

1. Run the install, edit the config, then start the service:

   ```sh
   deskflow install
   deskflow configure
   deskflow start
   ```

   By default, `deskflow install` follows the `latest` public channel. Public release channels are also available by upstream version, such as `deskflow install v1.26.0`.

   Preview builds require a subscription to [patreon.treyturner.info](https://patreon.treyturner.info) as verified through OAuth login. Sign-in will be triggered automatically if a login session doesn't exist when installing the preview channel:

   ```sh
   deskflow install preview
   ```

1. Optional: autostart the service on interface up

   ```sh
   deskflow enable-autostart

   # If you change your mind later, run:
   #deskflow disable-autostart
   ```

When installed, Deskflow lives under `/media/fat/Scripts/deskflow-dist/`.

## `deskflow` commands

Run `deskflow help` at any time for built-in usage text.

**Only `install`, `preview-*`, and `help` are available before Deskflow is installed.** The remaining commands appear afterwards.

| Command                 | What it do 👈                                                                    |
| ----------------------- | -------------------------------------------------------------------------------- |
| `install [CHANNEL]`     | Download and install Deskflow. Re-running it reinstalls or switches channel.     |
| `update`                | Check the currently installed channel and update if a newer build is available.  |
| `start`                 | Start the MiSTer Deskflow client in the background.                              |
| `run`                   | Run the MiSTer Deskflow client in the foreground with console output.            |
| `restart`               | Restart the MiSTer Deskflow client.                                              |
| `stop`                  | Stop any running Deskflow client process.                                        |
| `configure`             | Open the config file in a text editor such as `nano` or `vi`.                    |
| `trust <FINGERPRINT>`   | Trust a Deskflow server fingerprint.                                             |
| `untrust <FINGERPRINT>` | Remove a trusted Deskflow server fingerprint.                                    |
| `set-trust-mode <MODE>` | Persist `strict`, `tofu`, or `promiscuous` for future starts and autostarts.     |
| `show-trust`            | Show the current trust mode and trusted fingerprints.                            |
| `enable-autostart`      | Install network hooks so Deskflow starts automatically when networking comes up. |
| `disable-autostart`     | Remove the autostart network hooks.                                              |
| `remove`                | Stop Deskflow, remove autostart hooks, and delete the installed files.           |
| `status`                | Show channel, version, service state, and bundle info.                           |
| `preview-login`         | Sign in with Patreon and verify preview access with the brokerage.               |
| `preview-logout`        | Revoke and clear the preview brokerage session.                                  |
| `preview-doctor`        | Show preview brokerage status and diagnostics.                                   |
| `help`                  | Show command help.                                                               |

## Configuration

After install, your working config file is:

```text
/media/fat/Scripts/deskflow-dist/deskflow.conf
```

The installer creates it from the shipped example on first install; later installs/upgrades will preserve it.

```ini
[client]
remoteHost=192.168.1.100             # CRITICAL: must match the ip or hostname of your server
languageSync=false                   # only true if debugging keyboard layout issues

screenWidthFallback=640              # used until MiSTer core geometry is detected
screenHeightFallback=480             # used until MiSTer core geometry is detected

absoluteMovementSlackFactor=1.5 # absolute-mode slack factor around the real framebuffer

absoluteMovementScaleFactorX=0.5     # scale absolute-generated X motion
absoluteMovementScaleFactorY=0.5     # scale absolute-generated Y motion
absoluteMovementScaleLowThreshold=1  # absolute motion at/below this uses no dampening
absoluteMovementScaleHighThreshold=8 # absolute motion at/above this uses full configured scale

relativeMovementScaleFactorX=0.5     # scale direct relative X motion
relativeMovementScaleFactorY=0.5     # scale direct relative Y motion
relativeMovementScaleLowThreshold=1  # relative motion at/below this uses no dampening
relativeMovementScaleHighThreshold=8 # relative motion at/above this uses full configured scale

clipboardPasteHotkey=Super+v         # paste server clipboard text into the active core
clipboardPasteDelayMs=0              # delay between pasted text keypresses
clipboardPasteMaxChars=4096          # refuse larger text pastes; 0 disables this limit

screenshotClipboardEnabled=true      # publish new MiSTer screenshots to the server clipboard
screenshotClipboardDirectory=/media/fat/screenshots # watched screenshot root

[core]
computerName=mister       # must match a screen on server-side layout
port=24800                # only change if your server listens on a non-standard port

[security]
tlsEnabled=true           # set false if your server has TLS disabled
```

These are the settings most users will want to review:

- Networking
  - `remoteHost`: **You almost certainly need to change this** to the IP address or hostname of your Deskflow server.
  - `computerName`: The MiSTer screen name. This must match a screen name configured in your Deskflow server layout.
  - `port`: Leave this at `24800` unless your Deskflow server uses a different port.
- Desktop geometry
  - `screenWidthFallback` and `screenHeightFallback`: Startup fallback size used until Deskflow identifies the loaded MiSTer core from `/tmp/CORENAME`. If the loaded core is not in Deskflow's geometry table, the current geometry remains in use. Geometry changes are sent to the server automatically.
  - `absoluteMovementSlackFactor`: Absolute mouse mode reports a larger virtual screen to the server by this factor, creating a forgiving edge margin around the real MiSTer framebuffer. Relative mouse mode always reports the exact framebuffer size. Valid range is `1.0` to `5.0`.
  - `absoluteMovementScaleFactorX/Y` and `relativeMovementScaleFactorX/Y`: Scale emitted mouse movement without changing the screen geometry reported to the server. Absolute scaling applies to relative uinput deltas generated from absolute Deskflow positions; relative scaling applies to direct relative movements. Valid range is greater than `0.0` through `1.0`, and all default to `0.5`.
  - `absoluteMovementScaleLow/HighThreshold` and `relativeMovementScaleLow/HighThreshold`: Tune adaptive dampening. Movement at or below the low threshold uses no dampening; movement at or above the high threshold uses the configured X/Y scale. Raise the high threshold to keep small-motion boost active longer, or lower it to reach full dampening sooner.
- Clipboard paste (text) functionality
  - `clipboardPasteHotkey`: Hotkey used to type server clipboard text into the active core. Defaults to `Super+v`; supports any combination of `Control`, `Alt`, `Shift`, `Super`, and `Meta` plus one key. Set it empty to disable MiSTer paste.
  - `clipboardPasteDelayMs`: Milliseconds to wait between pasted text keypresses. Defaults to `0`, which emits as quickly as the uinput backend can send complete keypresses.
  - `clipboardPasteMaxChars`: Maximum text length accepted for a MiSTer paste. Larger server clipboard text is refused instead of typed. Set to `0` for unlimited paste length.
- MiSTer screenshot export functionality
  - `screenshotClipboardEnabled`: When `true`, Deskflow watches MiSTer's screenshot directory and publishes newly written screenshots as bitmap clipboard data to the server.
  - `screenshotClipboardDirectory`: Root directory watched for MiSTer screenshots. Defaults to `/media/fat/screenshots`; change it only if your MiSTer setup writes screenshots elsewhere.

## TLS and verification

By default, Deskflow expects TLS to be enabled on the server. Both `deskflow start` and `deskflow run` launch the client with the saved trust mode, defaulting to `tofu` ([trust on first use](https://en.wikipedia.org/wiki/Trust_on_first_use)). The first successful TLS connection will be automatically trusted and its fingerprint saved automatically.

If your Deskflow server has TLS disabled entirely, add this to `/media/fat/Scripts/deskflow-dist/deskflow.conf`:

```ini
[security]
tlsEnabled=false
```

After that, start Deskflow normally with:

```sh
deskflow start
```

If your server still uses TLS but you want encrypted connections without fingerprint verification, set the persisted launch mode to `promiscuous`:

```sh
deskflow set-trust-mode promiscuous
deskflow start
```

Use `promiscuous` only if you understand the security tradeoff. It keeps encryption enabled, but it does not verify that you are talking to the expected server.

To go back to the default trust-on-first-use behavior:

```sh
deskflow set-trust-mode tofu
```

If you want strict verification and only want to allow already-trusted server fingerprints:

```sh
deskflow trust <sha256-fingerprint>
deskflow set-trust-mode strict
```

To remove a previously trusted server fingerprint:

```sh
deskflow untrust <sha256-fingerprint>
```

You can check the current setting and trusted fingerprints at any time with:

```sh
deskflow show-trust
```

The selected mode is stored in:

```text
/media/fat/Scripts/deskflow-dist/.trust-mode
```

`deskflow start`, `deskflow run`, and autostart all respect the saved mode.

## Clipboard sharing

As MiSTer-deskflow is unable to hook the actual clipboard of a running core (if one even exists), traditional clipboard sharing isn't an option. The channel has been reused however for different but useful implementations:

### Text paste into MiSTer core

Text copied on the Deskflow server can be pasted into MiSTer with `clipboardPasteHotkey` (default `Super+V`) if clipboard sharing is enabled on the server. See options `clipboardPasteHotkey`, `clipboardPasteDelayMs`, and `clipboardPasteMaxChars`.

### Screenshot export

When `screenshotClipboardEnabled=true`, new `.png` or `.bmp` files under `screenshotClipboardDirectory` are also published back to the server clipboard as bitmap data. Use MiSTer's normal screenshot hotkey: `Super+PrtScr` for capture from the scaler and `Shift+Super+PrtScr` for capture from the framebuffer. Deskflow only watches the files MiSTer writes and does not trigger screenshots itself.

Deskflow's bitmap clipboard format is uncompressed, so screenshots can be much larger than their PNG files. **The Deskflow server defaults to a 3 MiB clipboard sharing limit which is likely to reject scaled HD screenshots.** Set the size limit to at least `8 MB` for 1920x1080 screenshots and `16 MB` for larger, or set `screenshotClipboardEnabled=false` if you do not want screenshot clipboard uploads.

## Usability tips

- You can lock the mouse (and keyboard) to the active screen using **Scroll Lock** (default), or remap as needed.
- In the server-side config, you can configure:
  - Delays before switching
  - Dead corners of different sizes to help accessing hidden taskbars
  - Translation of modifier keys like Ctrl/Alt/Shift between screens
  - Other custom hotkeys

## Known issues

### Virtual screen limitations

An inherent limitation of substituting an actual window manager with a virtual screen is that it can de-sync from the active core. This manifests like an invisible wall on some axis that the core cursor won't move past. It happens more often when using high mouse velocities, but the behavior can be exhibited more simply:

1. Move the mouse to the center of the MiSTer screen
2. Open the MiSTer OSD
3. Move the mouse consistently in any one direction for a stretch
4. Close the MiSTer OSD
5. Try to move in the same direction. Doh! 🤦‍♂️ Invisible wall.

This is caused by the virtual screen tracking input that is being intercepted by the OSD instead of sent to the core.

This happens only when Deskflow is sending **absolute** mouse movements. To send **relative** mouse movements, you must **both**:

1. Check **'Use relative mouse movements'** in the 'Advanced' tab of the Deskflow server's settings [as outlined in SERVER_SETUP.md](SERVER_SETUP.md#use-relative-mouse-movements), AND
2. **Lock the mouse to the MiSTer screen** using Scroll Lock (or the configured hotkey)

While this behavior only affects absolute movements, locking the cursor to a screen isn't always feasible. <a id="mitigations"></a>This fork therefore implements some further options to mitigate impact.

When receiving absolute movements, an `absoluteMovementSlackFactor` from `1.0` to `5.0` (default `1.5`) is applied to the core's native resolution, creating a margin around the screen. This helps avoid invisible walls at a cost of eventually needing to move back through any jumped values in the opposite direction, re-syncing the axis. You can think about the core desktop as being ["pan and scan"](https://en.wikipedia.org/wiki/Pan_and_scan) inside this space, and must balance the desire to avoid invisible walls with your willingness to re-sync at the other side.

When the cursor is locked to the MiSTer screen and relative movements are being sent, this factor is automatically reduced to `1.0` resulting in the core's native resolution.

The `absoluteMovementScaleFactorX/Y` and `relativeMovementScaleFactorX/Y` settings are separate from slack. Slack changes the absolute-mode coordinate space reported to the server, while scale factors reduce the relative uinput movement emitted to MiSTer. Each axis may be configured independently from greater than `0.0` through `1.0`, to taste and based on the speed of your mouse as used on other screens. The low/high threshold settings shape how quickly small movements ease into that configured dampening.

### Other issues & limitations

#### Keyboard support

Keyboard support is currently US-biased through nature of my inexperience with other keymaps. If you use a Deskflow server and/or keyboard in other languages or layouts and experience issues (likely), I'd appreciate you reporting them so I can offer better support.

#### Mouse wheel

Mouse wheel is not (yet?) implemented; it may not be possible or ultimately be limited.

## Troubleshooting

For live console output while connecting or debugging startup issues, use:

```sh
deskflow run
```

If Deskflow fails immediately on startup, check that your MiSTer exposes a writable `uinput` device:

```sh
ls -l /dev/uinput /dev/input/uinput 2>/dev/null
test -w /dev/uinput || test -w /dev/input/uinput
```

If neither path exists or is writable, Deskflow won't be able to inject keyboard and mouse events.
