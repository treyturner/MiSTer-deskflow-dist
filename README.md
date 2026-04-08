# MiSTer Deskflow

Deskflow lets you use the keyboard and mouse from one computer to control another over the network. This distribution packages Deskflow for MiSTer FPGA so you can control retro computer cores over the network from your regular-use system without the hassle or latency of a physical KVM.

This is a __client-only__ port. It requires you set up and run a Deskflow __server__ on a Windows, macOS, or Linux machine with the keyboard and mouse you intend to use.

## Before you start

Set up a Deskflow server on the computer with your keyboard and mouse.

1. Download, install, and open [Deskflow](https://github.com/deskflow/deskflow/releases/latest).
2. Click `Use this computer's keyboard...`, then `Configure Server`.

    <img width="429" height="174" alt="Step 2: Main window" src="https://github.com/user-attachments/assets/7e5c08db-50f4-415d-b644-5732245bb145" />

3. Drag the screen icon in the top right to where your MiSTer display is in relation to your server's screen.

    <img width="447" height="318" alt="Step 3: Drag new screen into layout" src="https://github.com/user-attachments/assets/6dbc727e-5064-425f-b0c2-c92b4a22e2be" />

4. It will initially be called `Unnamed`. Double-click to edit.

    <img width="447" height="318" alt="Step 4: Double-click 'Unnamed'" src="https://github.com/user-attachments/assets/f22ac99a-a8b5-42c8-a6d0-f8409182bc65" />

5. Change the name to `mister`, then click 'Save'.

    <img width="272" height="382" alt="Step 5: Rename to 'mister' and click 'Save'" src="https://github.com/user-attachments/assets/dec0f338-484f-4a7d-8103-d15121f92e5c" />

6. Now click the `Advanced` tab.

    <img width="447" height="322" alt="Step 6: Click 'Advanced'" src="https://github.com/user-attachments/assets/d1475df5-76b4-4db7-9ea2-b7bf998e0b11" />

7. Check `Use relative mouse movements` then click `OK`.

    <img width="446" height="317" alt="Step 7: Check 'use relative mouse movements'" src="https://github.com/user-attachments/assets/cb2d6b5c-b1de-4a87-ad8a-21b239479bb0" />

8. Note the IP of your server as it's needed for MiSTer-side setup, then click `Start`.

    <img width="429" height="174" alt="Step 8: Note IP and click 'Start'" src="https://github.com/user-attachments/assets/afeb8606-42b9-492a-b2a3-787fce2fd3fe" />

For more detailed server-side setup documentation, see:

- <https://deskflow.org>
- <https://github.com/deskflow/deskflow/wiki>

## MiSTer client installation

Download `deskflow.sh` into `/media/fat/Scripts` and make it executable:

```sh
cd /media/fat/Scripts \
&& curl -L -o deskflow.sh \
  https://raw.githubusercontent.com/treyturner/MiSTer-deskflow-dist/main/deskflow.sh \
&& chmod +x deskflow.sh
```

Run the install, edit the config, then start the service:

```sh
./deskflow.sh install
./deskflow.sh configure-from-cli
./deskflow.sh start
```

Optionally enable auto-start, which hooks `if-up.d` and `if-down.d`:

```sh
./deskflow.sh enable-autostart

# If you ever change your mind, run:
#./deskflow.sh disable-autostart
```

After installation, Deskflow lives under:

```text
/media/fat/Scripts/deskflow/
```

## `deskflow.sh` commands

Run `./deskflow.sh help` at any time for built-in usage text.

Before Deskflow is installed, only `install` and `help` are available. The rest of the commands appear after the first successful install.

| Command                    | What it do 👈                                                                    |
| -------------------------- | -------------------------------------------------------------------------------- |
| `install [CHANNEL]`        | Download and install Deskflow. Re-running it reinstalls or switches channel.     |
| `update`                   | Check the currently installed channel and update if a newer build is available.  |
| `start`                    | Start the MiSTer Deskflow client in the background.                              |
| `run`                      | Run the MiSTer Deskflow client in the foreground with console output.            |
| `stop`                     | Stop any running Deskflow client process.                                        |
| `configure`                | GUI configuration. (not yet implemented)                                         |
| `configure-from-cli`       | Open the config file in a text editor such as `nano` or `vi`.                    |
| `fingerprint-trust <MODE>` | Persist `strict`, `tofu`, or `off` for future starts and autostarts.             |
| `show-fingerprint-trust`   | Show the current fingerprint trust mode.                                         |
| `enable-autostart`         | Install network hooks so Deskflow starts automatically when networking comes up. |
| `disable-autostart`        | Remove the autostart network hooks.                                              |
| `remove`                   | Stop Deskflow, remove autostart hooks, and delete the installed files.           |
| `help`                     | Show command help.                                                               |

## Configuration

After install, your working config file is:

```text
/media/fat/Scripts/deskflow/deskflow.conf
```

The installer creates it from the shipped example on first install; later installs/upgrades will preserve it.

```ini
[client]
remoteHost=192.168.1.100  # CRITICAL: must match the ip or hostname of your server
virtualScreenWidth=1440   # scale up/down to taste
virtualScreenHeight=1080  # scale up/down to taste
inputBackend=uinput       # always uinput on MiSTer
languageSync=false        # only true if debugging keyboard layout issues

[core]
computerName=mister       # must match a screen on server-side layout
port=24800                # only change if your server listens on a non-standard port

[security]
tlsEnabled=true           # set false if your server has TLS disabled
```

These are the settings most users will want to review:

- `remoteHost`: __You almost certainly need to change this__ to the IP address or hostname of your Deskflow server.
- `virtualScreenWidth` and `virtualScreenHeight`: The fixed virtual size used for mouse movement. If pointer movement feels wrong, this is the first thing to tune.
- `computerName`: The MiSTer screen name. This must match a screen name configured in your Deskflow server layout.
- `port`: Leave this at `24800` unless your Deskflow server uses a different port.

## TLS and verification

By default, Deskflow expects TLS to be enabled on the server. Both `deskflow.sh start` and `deskflow.sh run` launch the client with the saved fingerprint-trust mode, defaulting to `tofu` ([trust on first use](https://en.wikipedia.org/wiki/Trust_on_first_use)). The first successful TLS connection will be automatically trusted and its fingerprint saved automatically.

If your Deskflow server has TLS disabled entirely, add this to `/media/fat/Scripts/deskflow/deskflow.conf`:

```ini
[security]
tlsEnabled=false
```

After that, start Deskflow normally with:

```sh
./deskflow.sh start
```

If your server still uses TLS but you want encrypted connections without fingerprint verification, set the persisted launch mode to `off`:

```sh
./deskflow.sh fingerprint-trust off
./deskflow.sh start
```

Use `--fingerprint-trust off` only if you understand the security tradeoff. It keeps encryption enabled, but it does not verify that you are talking to the expected server.

To go back to the default trust-on-first-use behavior:

```sh
./deskflow.sh fingerprint-trust tofu
```

If you want strict verification and only want to allow already-trusted server fingerprints:

```sh
./deskflow.sh fingerprint-trust strict
```

You can check the current setting at any time with:

```sh
./deskflow.sh show-fingerprint-trust
```

The selected mode is stored in:

```text
/media/fat/Scripts/deskflow/.fingerprint-trust
```

`deskflow.sh start`, `deskflow.sh run`, and autostart all respect the saved mode.

## Known issues

- Clipboard sharing isn't yet implemented but I believe will be possible
- Mouse wheels aren't yet implemented and support may be ultimately limited

## Troubleshooting

For live console output while connecting or debugging startup issues, use:

```sh
./deskflow.sh run
```

If Deskflow fails immediately on startup, check that your MiSTer exposes a writable `uinput` device:

```sh
ls -l /dev/uinput /dev/input/uinput 2>/dev/null
test -w /dev/uinput || test -w /dev/input/uinput
```

If neither path exists or is writable, Deskflow won't be able to inject keyboard and mouse events.

## Why fork?

Upstream Deskflow is built for desktop operating systems. MiSTer is a much leaner, headless ARM Linux environment. Getting Deskflow running therefore requires a number of platform-specific changes.

- Deskflow targets 64-bit architecture. Some bugs around this required fixing, there may be more, and I'm not remotely interested in upstream politics until/unless I know the changes I'm suggesting are rock-solid.
- As MiSTer doesn't provide an X11 or Wayland desktop session, this port implements a new backend.
- Input has to be injected as synthetic keyboard and mouse devices so the active MiSTer core can see it.
- The stock MiSTer Linux image is older than the runtime expected by a modern Deskflow build, so this package ships the needed runtime libraries, OpenSSL modules, and XKB data alongside `deskflow-core`.
- Mouse movement is handled through a fixed virtual screen size from the config file since there's no window system for querying display geometry.
- Configuration is script-driven as the GUI can't be built without a real windowing system.

For these reasons and for efficiency in distributing updates, releases are delivered as a bootstrap script and runtime bundle instead of a single standalone binary.
