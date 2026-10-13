# DEEBOT OZMO T8 AIVI root notes (Rockchip PX30, class `bs40nz`)

Notes from rooting my own DEEBOT OZMO T8 AIVI (UK retail, 2020). Its cloud class reports as
`bs40nz`, which `deebot_client` aliases to the global T8 AIVI `x5d34r`. It's built on a Rockchip
PX30 (quad Cortex-A35), firmware `fw_arm 1.11.8`, kernel `4.4.185 aarch64`. Userland is BusyBox
plus an Ecovacs ROS stack (see Architecture). The `product` key is `AI_px30` and the camera sensor
is a SOI JX-H65 over MIPI.

Most of this corrects or extends what's already around: the 37C3 `eco_passwd` formula is wrong for
px30 (three ways, below), and the `:8888` service is usually misidentified, so I pin down what it
actually is on this model. Credit to Dennis Giese / the dontvacuum.me research for the UART
groundwork, and to itsjfx's X1 notes.

## Debug port and hardware

No teardown needed. There's a JST PHDR-08VS 8-pin header under a rubber plug beside the
TrueMapping lidar, reachable with the lid up. It carries UART, 3V3 and SWD, and there's a
micro-USB in the same bay for the Rockchip USB download path. Some units have the header rotated
180 degrees, so meter GND (against the micro-USB shell) before connecting.

Use a 3.3 V UART adapter only, never 5 V. 115200 8N1 works; U-Boot may run at 1500000. U-Boot has
`bootdelay=0`, so mash Ctrl-C from power-on. Board id is `evb_px30`.

## Root: the eco_passwd algorithm (the 37C3 slide is wrong for px30)

The root password is unique per unit and set at every boot by the obfuscated `eco_passwd`
(XOR-0x55). I captured the real algorithm on-device and reproduced it offline:

```
input   = product + MAGIC + sn[-8:] + "\n"
product = "AI_px30"                              # the "product" key in /etc/fw.manifest, not the market class
MAGIC   = "d4:3d:7e:fa:12:5d:C8:02:8F:0A:E2:F5"  # ASCII literal, mixed case exactly as shown
sn[-8:] = last 8 chars of the 20-char serial     # e.g. "....HENE0005"
pw      = base64( sha256_hex(input) + "  -\n" )[:8]
```

In other words the binary runs `echo "<product><MAGIC><sn8>" | sha256sum | awk '{print $1}' | base64`,
takes the first 8 chars, then `echo "root:<pw>" | busybox chpasswd`. The user is `root`, with no
serial lockout. The firmware reads the SN from the `sys` partition at offset 1 MiB, and it matches
the printed/dustbin SN.

Three ways the 37C3 slide-79 formula differs on px30:

1. `product` is `"AI_px30"`, not the market class or model string.
2. Only the last 8 characters of the serial are used, not the full 20.
3. It's base64 of the ASCII `sha256sum` text (hex digest, two spaces, `-`, newline), truncated to
   8. Not base64 of the raw digest.

## Flash layout and persistence

GPT, 512-byte LBAs:

| part | LBA | notes |
|---|---|---|
| uboot | 0x4000 | |
| trust | 0x6000 | |
| mx | 0x8000 | calibration (floats), don't touch |
| my | 0xa000 | calibration, don't touch |
| sys | 0xc000 | SN at 1 MiB |
| boot1 | 0xd000 | Android boot image |
| rootfs1 | 0x10000 | squashfs, read-only (`eco_passwd` lives here) |
| boot2 | 0x33000 | |
| rootfs2 | 0x36000 | |
| data | 0x59000 | ext4, read-write |

Read-only dumps over `rockusb`/`rkdeveloptool` are fine, but writes must use the U-Boot
`ums 0 rknand 0` command, which exposes the whole NAND via the FTL as a USB mass-storage disk.
`rockusb` IDB mode only reaches below LBA 0x10000, so the data partition at 0x59000 is unreachable
that way (reads above it return `0xCC` filler).

For persistent root without repacking the rootfs: `/etc/rc.d/autostart.sh` runs every
`/data/autostart/*.sh` as root at boot, after `eco_passwd`. Drop a script there that sets a known
password or starts dropbear/telnetd. `/data` (ext4) is writable by design, so this is fully
reversible (delete the file) and avoids the only real brick risk, which is a rootfs repack. On
Windows, remove Zadig's WinUSB binding for `2207:330D` (`pnputil /delete-driver`) or the `ums` disk
won't mount as storage.

## Camera (local, no cloud)

The `rkisp1` ISP pipeline is live, with the JX-H65 sensor over MIPI. `/dev/video0` is
`rkisp1_mainpath` at 1280x960 YUYV (V4L2 multiplanar). There's no hardware MJPEG; the node only
offers raw formats.

Sensor exposure and gain live on `/dev/v4l-subdev2`: `exposure` (4-1000), `gain` (0-16383, digital,
weak effect) and `analogue_gain` (0-127, the dominant control). A fresh `STREAMON` resets the gain
low, so the first frames are near-black.

Capture does not require stopping the stock vision daemon. Nothing holds `/dev/video0` open
continuously, even during a clean, so you can grab or stream frames as root while the robot runs.

The app's "live video" is NetEase/Alibaba cloud WebRTC. There's no local RTSP, so the V4L2 path
above is the only local option.

One quirk worth knowing: while cleaning in the dark the robot strobes a visible white LED at
roughly 1 Hz so its obstacle recognition can see. On a naive capture these land as fully blown-out
frames, or partially blown bands with a rolling shutter. The stock app discards them, and a
streamer should too (drop frames whose mean luma or saturated-pixel fraction spikes).

I put together a small static-aarch64 streamer (V4L2, libjpeg, HTTP MJPEG) with a fisheye dewarp
(equidistant; the content is much wider than it is tall, so output 16:9), a brightness-targeting
auto-exposure loop on `analogue_gain`, and the flash-drop above. It gives a clean local FPV feed. I
can share the source if it's useful.

## Software architecture (for local control work)

`eros` is Ecovacs's ROS Melodic stack: a real `roscore` master on `127.0.0.1:11311`, with nodes
loaded by `/usr/bin/deebot` (`eros_node_*`: task_manager, slam, clean, hardware_platform and so on)
over an Ecovacs message bus called medusa. The CLI is `mdsctl <element> '<json>'`, talking to
`/tmp/mds_cmd.sock`.

App commands flow `mqtt -> bumbee -> switch` (which uses `/etc/conf/func_table.conf`)
`-> wifihandler` (`/usr/lib/WHmethods`) `-> rosnode -> ROS nodes`.

Manual drive is the app command `move` (func_table `clean_logic`), which becomes the ROS message
`task/RemoteMove{ uint8 type, uint16 lastTime, int16 v, int16 w }`. The `type` enum is 0=FWD,
1=BACK, 2=STOP, 3=BRAKE, 4=SPINLEFT, 5=SPINRIGHT, 6=GOTO, 7=ROTATE, 8=REVERSE, 9=CUSTOM,
10=TURN_LEFT, 11=TURN_RIGHT. The handler reads `act` (one of forward/backward/spinLeft/spinRight/
turnAround) plus `angle`, and `lastTime` is a dead-man timer (it auto-stops unless refreshed).
Driving it cleanly from a local root shell is still unsolved for me: `mdsctl wifihandler` injection
is accepted but doesn't actuate, and `hardware_platform` subscribes a low-level `WheelSpeed` topic
that bypasses the safety ramp. If anyone cracks the local drive path, I'd like to hear it.

One warning: `medusa.sh` and `deebot.sh` are cross-coupled supervisors. Each one `killall`s the
other when its process exits, so a plain `kill medusa` starts a mutual-restart storm. Restart via
the `/tmp/*_exit.mark` path, or just reboot.

## The `:8888` service on this model (security note)

Various notes mention a goahead server on `:8888` with hardcoded creds. On this px30 unit it's
GoAhead (`/usr/sbin/goahead`, started by `/etc/rc.d/goahead.sh`), serving the WiFi-provisioning and
factory-config API from `/etc/www` (CGI `reqDo`). It binds `http://*:8888`, so it's on all
interfaces and reachable across the LAN, and the process runs as root. On some other models the
medusa `bumbee` plugin serves `:8888` instead; here bumbee's web server is present but disabled in
`deebot_px30_ai.conf` (`web_enable:0`), so goahead owns the port.

The security problem: it's protected only by a default/hardcoded credential, and past that the API
exposes device identity, a stored-credential readout, and WiFi/factory configuration, and it shells
out through `popen()`. That makes `:8888` a remotely reachable, root-privileged surface, which is a
serious exposure on any unit left on a routable network.

If you own one of these, disable or firewall `:8888` on your network and change the stored
credential. I'm keeping the remote-root specifics (exact credential, endpoint, injection) out of
this writeup; those belong in coordinated disclosure to the vendor rather than a public PoC.

---

Everything device-specific (serials, credentials, network details) is deliberately left out.
