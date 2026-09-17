# janus-gate — project handoff

An ESP32-S3 controller that drives two gate motors, answers to Home Assistant,
and keeps working when the network does not.

This is the complete context: what the device does, why it is built this way,
and — most usefully — every trap that cost real debugging time. The
architecture is ordinary; the failures were not. **Section 7 is the part worth
keeping.**

The YAML is the source of truth. Where this document disagrees with the config,
the config wins.

---

## 1. What it is

One wall-mounted box, one USB-C supply, one Wi-Fi connection. It replaces two
battery-powered RF remotes, a separate RF bridge, and the pile of hardware that
would otherwise live on that wall.

It drives two independent motors by two completely different mechanisms.

**GARAGE.** The original 3 V remote lives inside the enclosure. Its battery is
gone; it runs off the ESP32's 3.3 V rail. The ESP32 shorts the remote's own
button contacts through optocouplers — electrically identical to a finger
press. The drive therefore sees a remote it was already paired with. No
protocol was reverse-engineered and no gate security was bypassed. This matters
because the garage remote turned out to be a rolling-code unit that cannot be
cloned at all.

**GATE.** A CC1101 433 MHz transceiver learns the second remote's raw OOK frame
**on the device itself**, stores it in NVS, and replays it. Nothing is baked
into firmware.

Both paths are fully local. Neither Wi-Fi nor Home Assistant sits in the
control chain — the physical toggle switches work with the network down, which
is the entire point of the device.

`docs/img/enclosure-interior.jpg` shows the interior: DevKit on perfboard, the
two PC817 modules, the CC1101 with its SMA antenna.
`docs/img/front-panel.jpg` shows the finished panel — two ON-OFF-ON rockers
labelled *Garaż* and *Brama*.

For the public-facing overview see [README.md](README.md); this document is the
maintainer's record and owns the complete trap list, the verification status
and the outstanding work.

---

## 2. How the design got here

Worth recording, because several decisions look arbitrary without the history.

**The original plan had no CC1101.** Both remotes were to be driven through
optocouplers. The gate remote runs on 12 V, which would have meant a boost
converter and a second power path — so the plan became: replace that remote
with an RF transmitter replaying its code.

**Then the CC1101 was cut, then reinstated.** A Sonoff RF Bridge already in
Home Assistant could fire the gate, so for a while the design routed the
physical button through HA. That was abandoned once it became clear it made the
gate depend on Wi-Fi, the router and the HA server — three things that can
fail, in a device whose purpose is to work when they do.

**Learning moved onto the device.** The gate remote lives at the gate, 30 km
away, and was never available at the bench. Rather than make two trips, the
firmware grew a learn mode driven from a phone over the ESP's own access point.
The side effect is a better product: it can be re-taught, moved to another
gate, or taught an extra remote with no reflash.

**The garage remote could never be cloned**, which is why it is wired through
optocouplers rather than replayed. The RF bridge failed on it, it costs around
80 PLN where simple remotes cost 10, and its PCB is dated 2024 — all consistent
with rolling code. The optocoupler approach sidesteps that entirely: the remote
does its own rolling-code work, the ESP just presses its buttons.

---

## 3. Repo layout

The config began as a single 1003-line `janus-gate.yaml`. It is now split by
concern, one package per file. Nothing about the device changed in the split —
it was verified by diffing the expanded configuration before and after.

```
janus-gate.yaml          98   production entry: identity, tuning knobs, package map
janus-gate-debug.yaml    90   diagnostic entry: same packages + receiver override
packages/
  core.yaml              60   esphome, esp32, logger, api, ota, wifi, web_server
  radio.yaml             90   SPI, CC1101, remote_transmitter, Ebyte pinout
  garage.yaml           290   optocoupler path, end to end
  gate.yaml             280   CC1101 replay path, end to end
  learning.yaml         330   remote_receiver + on_raw, learned codes, learn UI
  diagnostics.yaml       35   device-level only: Wi-Fi, uptime, IP, state reset
  debug.yaml             66   radio sniffing aids — debug builds only
secrets.sample.yaml           template; copy to secrets.yaml (gitignored)
.idea/runConfigurations/      four shared JetBrains run configurations
```

Each drive file reads top to bottom as a complete control path: outputs,
scripts, buttons, switch, assumed state. All tuning substitutions live in the
production entry file so they are visible in one place on arrival.

Two placements are deliberate and non-obvious:

- **`remote_receiver` lives in `learning.yaml`, not `radio.yaml`.** Its
  `on_raw` trigger *is* the learning logic, and the lambda works on the raw
  pulse vector `x`, which exists nowhere outside that trigger's scope. It
  cannot be separated, so it sits with the rest of learning.
- **The learned-code globals live in `learning.yaml`**, which is their sole
  producer. `gate.yaml` reads them by id and never writes them.

**Verifying a refactor.** `esphome config` prints the fully expanded
configuration. Capture it before a structural change, capture it after, and
compare the parsed trees with list order ignored. An identical tree proves the
change was behaviour-neutral. That is the reason the package split can be
trusted without reflashing to find out.

---

## 4. Hardware

### In the build

| Item | Notes |
|---|---|
| ESP32-S3 DevKitC-1 WROOM-1 N16R8 | 16 MB flash, 8 MB **octal** PSRAM |
| 2 × PC817 2-channel optocoupler module | 4 channels used |
| CC1101 433 MHz — **Ebyte E07-M1101D-SMA** | 1 used, 1 spare, SMA antenna |
| 2 × ON-OFF-ON momentary toggle, DPDT | 6 pins each, one pole wired |
| Remote 1 "Garage" | 3 V, was CR2032. Four buttons: Stop / Open / Close / Lock |
| 100 nF + 47 µF, twice | one pair at the remote, one at the CC1101 |
| Perfboard 5 × 7 cm | everything soldered |
| 3D-printed enclosure | done, with printed labels |

### Present, unused

2 × AMS1117 3.3 V — the fallback if RF range disappoints and the radio needs
its own filtered rail. 1 spare CC1101, 2 spare optocoupler modules.

The spare CC1101 is genuinely useful for diagnosis: it is the only way to watch
the device's own transmission, since a single transceiver cannot hear itself
while sending.

### Not bought yet

RFID reader (stage 7). **PN532 over RC522** if the reader goes outside the
enclosure on a cable — RC522 is SPI-only and SPI gets unreliable past 20–30 cm.
PN532 speaks I²C and UART. Plus NTAG213 stickers for 3D-printed fobs.

---

## 5. Wiring

### Garage — optocouplers

| Function | GPIO | Wire | Module | Channel |
|---|---|---|---|---|
| Stop | 4 | grey | Opto 1 | 2 |
| Open | 5 | white | Opto 1 | 1 |
| Close | 6 | yellow | Opto 2 | 1 |
| Lock | 7 | orange | Opto 2 | 2 |

GPIO → IN+, GND → IN−. The transistor side goes to the remote's button pads:
collector to the pad at the higher potential, emitter to the lower. **The
transistor side needs no ground** — it shorts two contacts and has no idea what
potential they sit at.

### Physical switches

| Switch | Lever | GPIO |
|---|---|---|
| Garage | up | 15 |
| Garage | down | 16 |
| Gate | up | 17 |
| Gate | down | 18 |

Common to GND, internal pullups, no external resistors.

### CC1101 — Ebyte E07-M1101D-SMA

| Pin | Signal | ESP32-S3 |
|---|---|---|
| 1 | GND | GND |
| 2 | VCC | 3.3 V |
| 3 | GDO0 | GPIO 9 — transmit |
| 4 | CSN | GPIO 10 |
| 5 | SCK | GPIO 12 |
| 6 | MOSI | GPIO 11 |
| 7 | MISO | GPIO 13 |
| 8 | GDO2 | GPIO 8 — receive |

### Power

Single USB-C into the DevKit. The remote and the CC1101 each take **their own
branch from the DevKit's 3V3 pin** — star, not chain. Ground returns to one
point.

All the DevKit's 3V3 pins are the same node on the same regulator. Using
different pins does not create separate rails; it only separates the wire runs,
which is the actual point. A genuinely separate rail would need one of the
spare AMS1117s fed from 5 V.

### Pins to leave alone

26–32 flash · **33–37 octal PSRAM** (specific to N16R8) · 0/3/45/46 strapping ·
19/20 USB-JTAG · 43/44 UART0 · 38 or 48 RGB LED (board revision dependent) ·
14/21/42 reserved for RFID.

---

## 6. Design decisions worth preserving

### Raw timings, not decoded protocols

Learned codes are stored as raw pulse arrays (`int32_t[128]` plus a length),
not as a decoded rc_switch code. Decoded storage would be far smaller and
simpler, but only works when a built-in decoder recognises the protocol. Since
learning happens on site with no chance to debug, universality beat elegance.

### Two-frame confirmation

The receiver hears noise constantly, and noise bursts can be long enough to
look like a frame. Storing the first arrival would mean capturing junk before
the remote is even pressed.

So the first usable frame is held only as a *candidate*; a code is stored when
a second frame arrives matching it within tolerance (a quarter of each pulse
width plus a flat 60 µs). Real remotes repeat; noise does not.

**Operationally: hold the remote button, do not tap it.**

This also resolved a design argument. Early on the instinct was to fight noise
at the radio layer — narrower bandwidth, higher filter. That turned out to be
exactly wrong (see section 7). With software confirmation in place, the correct
move is to let the receiver hear everything and filter in logic.

### `send_*` primitives vs `cmd_*` logic

`send_garage_open` pulses an optocoupler. `cmd_garage_open` carries direction
logic and calls the primitive.

Two reasons: it avoids recursion, and it keeps the raw actions reachable for
debugging — exposed as the `(direct)` diagnostic buttons, which bypass the
state machine entirely. Those buttons earned their place repeatedly.

The split also has a consequence that only shows up when extending the device.
The **programming button** (Stop and Open shorted simultaneously, to put the
drive into its programming mode) is its own primitive rather than a call to
`send_garage_stop` plus `send_garage_open` — because `send_garage_stop` is not
pure: it also stops the travel timer and clears `garage_dir`. A programming
press must not tell the device anything about where the door is.

### Anti-reverse: opposite direction while moving sends Stop

The garage drive **reverses instantly** on the opposite command, with no stop
in between. That is hard on the motor under load and almost never what the user
meant.

The device tracks assumed direction and substitutes a Stop. **The state is an
assumption, never a measurement** — there is no feedback from either drive, and
it drifts if someone uses the original remote or the door stalls.

That was accepted deliberately, because the failure modes are asymmetric:

- Stale "moving" → one wasted press; Stop on a stationary door does nothing.
- Stale "idle" → behaves exactly as if the feature did not exist.

Neither case is worse than not having the feature. That asymmetry is what makes
an unverifiable heuristic acceptable here, and it is the general test worth
applying to any similar shortcut.

`travel_time` is deliberately longer than real travel (45 s vs ~30 s) for the
same reason: too long costs a press, too short lets an opposite command reverse
the drive — the exact thing being prevented.

### The gate's anti-reverse needs a fallback the garage does not

The garage's Stop is a hard-wired contact and always exists. The gate's Stop is
a learned RF code that **may never have been learned** — that remote may not
even have a Stop button.

So `cmd_gate_*` checks `len_gate_stop > 0`. Without a Stop code it sends the
requested direction anyway and logs a warning. Doing nothing would be worse
than the reverse the logic exists to prevent.

### Lock is dangerous

On most drives "Lock" disables the motor and makes it ignore every remote until
unlocked. It is the one command that can lock the user out remotely, from a
phone. Distinct icon, warning comment, and it should be hidden or
confirmation-gated in Home Assistant.

---

## 7. The traps

Each of these cost hours. Several were misdiagnosed first, which is itself part
of the lesson.

A thread runs through several of them, and it is the most transferable thing in
this document: **an indicator tells you about what it measures, not about what
you want to know.** An LED that reports encoder power read as "transmitting".
An exit code that reports the toolchain ran read as "firmware built". A
`transmit_raw` that reports the action fired read as "a signal went out". Each
one was honest; each one was asked the wrong question.

### The remote had two power pads and only one was connected

**This is the big one.** It cost a 60 km round trip.

Early in the build the remote's metal battery tabs were squashed together and
shorting, which killed the ESP32 on connection — USB overcurrent protection
doing its job. Cutting the tabs out was the right call: they were useless with
the battery gone and would have shorted again once the enclosure closed.

But the positive tab had **two pads, electrically separate**. One feeds the
encoder and the status LED. The other feeds the RF output stage. Soldering to
only one left the transmitter with no power.

The remote then looked perfect: buttons responded, LED lit, voltage measured a
steady 3.2–3.3 V. It transmitted nothing at all — and had transmitted nothing
since the day it was wired in. This was only found after driving to the gate,
finding nothing worked, and reasoning backwards.

The diagnostic that would have caught it sooner is a receiver listening while
the button is pressed. That is what the debug build in section 10 exists for.

### CC1101 does not switch itself into TX

`remote_transmitter` will happily generate perfect timings on GDO0 while the
CC1101 sits in receive mode. Everything looks right — the action fires, the
logs are clean, the learned code is valid — and nothing reaches the air.

The two components have **no coupling whatsoever**. `remote_transmitter` keys
GPIO9 through the RMT peripheral; the radio's own state machine moves only on
explicit actions. In the ESPHome component, `begin_tx()` has exactly one
caller: the `cc1101.begin_tx` YAML action. `configure()` leaves the chip in RX
and there it stays.

The radio has to be told explicitly:

```yaml
- cc1101.begin_tx: radio
- remote_transmitter.transmit_raw: { ... }
- cc1101.begin_rx: radio
```

with `non_blocking: false` on the transmitter. Since ESPHome 2025.11.0
`non_blocking` defaults to `true`, so `transmit_raw` returns immediately and
`begin_rx` would yank the chip out of TX mid-frame. The alternative is
restoring RX from an `on_complete:` trigger, but the blocking form is safer:
`on_complete` leaves failure paths where the radio can be stranded in TX and
stop receiving entirely until a reboot.

`begin_rx` is as necessary as `begin_tx` — without it learning stops working
after the first transmission.

This is separate from the `gdo0_pin` trap below. `begin_tx` skips the gdo0
block when the pin is unset, so the fix and the trap coexist: **do not add
`gdo0_pin` to make this work.**

**Expected side effect.** With blocking transmission the log shows
`a scheduled task took a long time for an operation (~430 ms), max is 50 ms`
on every gate transmission. This is not a fault. Eight repeats of a ~44 ms
frame plus 10 ms gaps is around 430 ms, and ESPHome warns above 50 ms. The
ESP32 task watchdog is measured in seconds, so there is no reboot risk; the
device simply does not service the API or the switches for that fraction of a
second. The garage never produces this warning because `delay:` in a script
yields to the scheduler — only the blocking radio transmission holds the loop.

Useful corollary: **the absence of `PLL lock failed` in the log is positive
evidence that the chip entered TX.** `enter_calibrated_` logs that warning on
failure and retries three times before giving up with an error. Silence there
means calibration succeeded.

### `gdo0_pin` in the `cc1101:` block silently kills transmit

Setting it re-routes the pad away from the RMT peripheral. The chip enters TX,
the logs look fine, nothing is emitted.

The mechanism is worth knowing, because it explains why the trap is so
reliable: the component defers its pin-mode setup until after every other
component has finished `setup()`, with a source comment stating the intent
plainly — *"This handles the case where remote_transmitter runs after CC1101
and changes pin mode."* It is designed to win that race. **The config
deliberately omits it. Do not add it.**

### Optocoupler module jumpers must stay REMOVED

A channel conducted permanently, holding a remote button down, which locked up
the remote's encoder and made *every* button stop working. The remote looked
broken.

The per-channel jumpers select trigger polarity. Fitted, the channels are
permanently active. Nothing on the board says so.

### Over-filtering killed reception — twice

Attempting to quiet the noise floor, `filter_bandwidth` was dropped to 58 kHz
and `remote_receiver.filter` raised to 250 µs. **Reception stopped completely,
both times.**

- Cheap remotes use SAW resonators that drift hundreds of kHz off nominal. A
  58 kHz window simply misses them; wide bandwidth forgives the detuning.
- The remote's short pulses measured ~275–300 µs — *below* a 250 µs filter once
  jitter is accounted for. Every bit was discarded as a spike.

The second occurrence was only caught because the repo had a working commit to
diff against.

### Building under MSys/Mingw reports success and produces nothing

On Windows, running `esphome compile` from a Git Bash / MSys shell **deletes
the build directory, builds nothing, and exits 0 with
`INFO Successfully compiled program.`** The only hint is buried above it:

```
MSys/Mingw is no longer supported...
WARNING Firmware not found: janus-gate.bin
WARNING ELF not found: janus-gate.elf
```

The same command in PowerShell produces a full build with binaries. Both exit
0 and both print the success line.

**Build from PyCharm or PowerShell, never from an MSys shell**, and confirm
that `firmware.ota.bin` actually appeared rather than trusting the exit code.
This cost a set of previously-built artifacts.

### A toggle lever pivots

Pushing the lever **up** closes the contact on the **lower** side of the body.
GPIO assignments are by lever position as the user experiences it, which is the
opposite of terminal position. Fixed by swapping wires; if the connectors are
ever re-made, the inversion applies again.

### Ebyte pin 7/8 are swapped vs the generic module

Generic CC1101 breakouts put GDO2 on 7 and MISO on 8. **The Ebyte E07-M1101D
has them the other way round.** Wiring from a generic diagram gives a dead SPI
bus.

Reliable orientation: **pin 1 is GND and has continuity with the SMA connector
shell.** Beep from the shell to each pin.

### Wrong USB port on the DevKitC-1

Two USB-C sockets: one through a CP2102N bridge (UART), one native to the S3.
First flash **must** use the UART port; the native one is unreliable until
working firmware exists. Tell: Windows shows the native port as a generic "USB
Serial Device"; the UART port appears as a Silicon Labs CP210x.

### `dump: all` floods the log

It also runs the *infrared* decoders. Pronto and Beo4 will cheerfully
"recognise" pure RF noise and emit megabytes of hex. Raw replay needs no
decoder.

`dump: none` is **not valid** — there is no dumper by that name. Omit the
`dump:` line entirely. `on_raw` fires regardless.

### Noise can fake a confirmation

At one point a code was "CONFIRMED" at a plausible 50 pulses from what may have
been two agreeing noise bursts. The decisive test is free: arm learn mode, put
the remote out of reach, and wait out the full timeout. If nothing is stored,
the confirmation logic is sound.

This is the general shape — **every positive result needs a negative control.**
A remote known to work, held at the same distance, proves the receiver path. An
armed learn mode with no remote nearby proves the rejection path.

---

## 8. Load-bearing settings — do not "improve" these

Found empirically, verified working. Several look wrong and are not.

```yaml
cc1101:
  filter_bandwidth: 200kHz    # NOT 58kHz — narrow bandwidth kills reception
  rx_attenuation: 0dB         # noise is handled in software, not here

remote_receiver:
  filter: 100us               # NOT 250us — the remote's pulses are ~275us
  idle: 6ms

remote_transmitter:
  non_blocking: false         # required by the begin_tx/begin_rx sequence

api:
  reboot_timeout: 0s          # keep this even with HA connected
```

`pulse: 400ms` for the garage optocouplers is verified on this drive.

On `reboot_timeout`: the ESPHome default reboots the ESP every 15 minutes while
HA is unreachable. For a gate controller that turns a router outage into a
device stuck in a reboot loop. Keep it at zero.

---

## 9. Building and flashing

ESPHome 2026.9.0 in `.venv`. Four shared JetBrains run configurations live in
`.idea/runConfigurations/` and are tracked in git — `.gitignore` uses
`.idea/*` with a negation rather than `.idea/`, because git cannot re-include
anything below an excluded directory.

| Configuration | Command |
|---|---|
| ESPHome: flash over USB | `python -m esphome run janus-gate.yaml` |
| ESPHome: build OTA package | `python -m esphome compile janus-gate.yaml` |
| ESPHome: DEBUG flash over USB | `python -m esphome run janus-gate-debug.yaml` |
| ESPHome: DEBUG build OTA package | `python -m esphome compile janus-gate-debug.yaml` |

They run as Python *module* configurations against the project SDK, so they
pick up the `.venv` interpreter with no hardcoded path to an executable.

No device is hardcoded: ESPHome's interactive port chooser is left in place
because a fixed port breaks whenever the board re-enumerates, and because this
board has two USB sockets where the choice matters (see section 7).

**Before every flash: disconnect the remote from the optocouplers.** GPIO
states are undefined during boot and the garage can fire.

The OTA artifact lands at `.esphome/build/janus-gate/build/firmware.ota.bin`
and is uploaded through the device's own web panel — no laptop or toolchain
needed on site. **Production and debug share that path**, because the debug
build deliberately keeps the production device name. Rebuild the one you want
immediately before uploading, so you know which firmware you are shipping.

And see the MSys trap in section 7 before trusting any build.

---

## 10. The debug build

`janus-gate-debug.yaml` is the same device with the same packages; only the
receiver is opened up and sniffing aids are added. It exists to answer one
question: **is this remote transmitting anything at all?**

It overrides exactly two receiver keys — `filter` drops from 100 µs to 10 µs
and `dump: raw` is turned on — and adds `packages/debug.yaml`. The log fills
with received noise by design. The two-frame confirmation in `on_raw` survives
the merge, so learning still works while sniffing.

`filter_bandwidth` and `rx_attenuation` are **not** overridden: 200 kHz and
0 dB is already the most open the front end gets.

Buttons retune the radio between **433.92 and 433.42 MHz at runtime** via
`cc1101.set_frequency`, with a text sensor showing which is live. This matters
more than it looks: hearing nothing proves nothing unless the radio listens
where the remote talks, and a remote sold as "433 MHz" is usually but not
always 433.92. The 200 kHz window around 433.92 does not reach 433.42 at all.
`set_frequency` drops to IDLE, rewrites the FREQ registers and re-enters RX by
itself, so sniffing resumes with no further action.

**Other runtime knobs, if needed.** The component also exposes
`cc1101.set_modulation_type`, `set_symbol_rate`, `set_fsk_deviation`,
`set_filter_bandwidth`, `set_rx_attenuation` and `set_output_power` as actions.
If a remote is suspected of using FSK rather than OOK, that can be swept from
the panel without reflashing — though FSK demodulation also depends on
deviation and data rate, so it becomes a parameter search rather than a single
test.

**RSSI is not available** in this configuration. The component exposes it only
through the `on_packet` trigger, which requires FIFO packet mode; raw replay
runs in async serial mode. Signal strength would be the ideal instrument for
"is anything arriving", and it is out of reach. The substitute is watching
whether the noise floor *changes* when a button is pressed, since AGC responds
to received power regardless of modulation.

**How to use it**

1. Flash the debug build over USB.
2. Watch the log. With the filter this low it is noisy — that is the point.
3. Press the suspect remote close to the antenna. A real frame appears as a
   long structured burst, clearly distinct from short scattered noise.
4. Heard nothing? Press "DEBUG tune 433.42 MHz" and repeat. Silence on one
   frequency is not a verdict.
5. Run a positive control: a remote known to work, at the same distance. If
   that produces nothing either, the receiver is the problem, not the remote.
6. Flash `janus-gate.yaml` to go back.

---

## 11. Verified vs not

### Verified

- All four garage optocoupler channels fire the remote.
- Physical switches work with Wi-Fi down.
- Learning captures a real frame and rejects noise.
- **Transmit works end-to-end** — a learned code was replayed to a test 433 MHz
  device and it responded. This closed the project's longest-standing unknown.
- Home Assistant integration — done on site, native ESPHome integration, no
  config changes needed.
- The garage remote transmits again, after the second power pad was found.
- Production and debug firmware both compile clean: ~937 KB, RAM 34 %,
  flash 11.5 %.

### Still unverified

- **NVS persistence across a real power cut.** `int32_t[128]` globals with
  `restore_value: true` is a non-standard use of ESPHome's preference system. A
  reboot is not a sufficient test — pull the USB cable, wait, reconnect, check
  the pulse count. If it fails, the fallback is storing a decoded rc_switch code
  instead: tiny and certain, at the cost of only working for recognised
  protocols.
- **Whether the garage remote is still paired with the drive.** It transmits
  again, but it spent the whole build being pressed hundreds of times out of
  range. If it is rolling code, the counter has almost certainly drifted outside
  the receiver's resync window. Re-pairing at the drive is the likely next step
  — bring the drive's manual, because on some controllers the wrong sequence
  erases every stored remote, including ones that currently work. The
  programming button exists for exactly this.
- **Whether the gate's learned code opens the actual gate.** Replay is proven
  against a test device, not against that receiver.
- **Gate travel time.** `gate_travel_time: 45s` is a guess copied from the
  garage. Measure it.
- **Whether the gate drive reverses instantly.** All of `cmd_gate_*` assumes it
  behaves like the garage. If it stops by itself, the logic adds nothing and the
  switches can point back at `send_gate_*`.
- **The fallback AP path**, if it has not been rehearsed. Test it by setting a
  wrong SSID before travelling — it is the only route to the device in the
  field.

### Security, stated plainly

`web_server` has HTTP basic auth, but the ESP32 does no TLS here, so
credentials cross the network in clear text. That deters casual access on a
trusted LAN; it is not protection against someone already inside it.

The larger exposure is the fallback AP, visible from outside the building
whenever the main Wi-Fi is unreachable. `fallback_password` should be long and
random.

And the physical switches authenticate nobody. That is intentional and is the
whole point — but it means security ultimately reduces to who can reach the box
on the wall.

---

## 12. Learning a gate code on site

1. Power the device. Away from the home network it serves "Janus Gate
   Fallback".
2. Join that AP with a phone, browse to `192.168.4.1`, log in.
3. Press "Learn gate open". "Learn mode" shows `waiting for OPEN`.
4. **Hold** the matching button on the original remote, close to the antenna. A
   tap may not produce the two frames needed.
5. Watch "Learn mode" go `candidate held`, then `idle`.
6. Check the pulse count. A plausible fixed-code frame is roughly **25–100
   pulses**; around 50 is typical for 24 bits (two pulses per bit plus sync). A
   value of 2 or 4 means noise — repeat. A value near 128 means frames are
   merging and `idle` needs raising.
7. Repeat for close and stop.
8. Test with "Gate open (direct)", which bypasses the direction logic.

"Candidate pulses" is a live view of what is awaiting confirmation. If it keeps
jumping between unrelated values, the radio is chewing noise rather than
hearing the remote.

**Before travelling: press "Forget all gate codes."** Whatever is stored was
learned from a bench substitute.

---

## 13. Repo state and outstanding tasks

### Done

- Config split into packages, verified behaviour-neutral against the expanded
  configuration.
- The three stale comments corrected — `filter_bandwidth`, the receiver's
  `dump`/`filter` description, and the `reboot_timeout` advice — each rewritten
  to explain *why* the current value is right, so it does not get "improved"
  back to a version that failed.
- Secrets template renamed from `secrets.samle.yaml`, rewritten, and verified
  to cover exactly the seven secrets the firmware references. Tested by copying
  it into a clean directory as `secrets.yaml`: with only a generated `api_key`
  substituted, the configuration validates.
- The TX sequence documented at length beside `send_gate_*`, including why
  `non_blocking: false` is required.
- Git history audited for secrets: `secrets.yaml` was never committed, and none
  of the live secret values appear anywhere in history.
- JetBrains run configurations, production and debug.
- Garage programming button (Stop + Open together).

### Outstanding

1. **Two secrets are still template placeholders.** `ota_password` and
   `fallback_password` in the live `secrets.yaml` are still the literal strings
   from the sample file. The file is gitignored, so nothing leaked — but the
   values are guessable from the public template, and the fallback AP is
   reachable from outside the building. Generate real ones
   (`openssl rand -base64 24`) and reflash **before the repo goes public**.
2. **Never pushed.** `main` is many commits ahead of `origin/main`; the GitHub
   remote has seen none of this work. Decide public vs private first — item 1
   depends on the answer.
3. **Restore the lost logging** in `on_raw`. An earlier revision logged "Too
   short: N pulses" and "Too long: N pulses — frames may be merging". The second
   is diagnostic gold on site: it is the signal that `idle` is too short and
   repeated frames are merging, which makes learning fail while the radio works
   perfectly.
4. **`cover:` template entities.** Open, Close and Stop exist for both drives —
   exactly the `cover` shape. In HA that becomes one tile with arrows instead of
   a scatter of buttons, and it works properly with voice assistants and
   automations. No knowledge of real position is required.
5. **Hide or confirmation-gate Lock** in HA.
6. **README.** The traps in section 7 are more interesting than the
   architecture, and almost nobody documents them.
7. **`ota: password` vs encryption.** ESPHome warns that the OTA password costs
   roughly 3.5 KB of flash and that the API encryption key already
   authenticates uploaders. Worth switching.

`sniffer.yaml`, listed as clutter in an earlier handoff, does not exist in the
repo. Nothing to do.

### Do not

- Do not "optimise" the radio settings in section 8.
- Do not add `gdo0_pin` to the `cc1101:` block.
- Do not remove `cc1101.begin_tx` / `begin_rx` from `send_gate_*`.
- Do not merge `send_*` and `cmd_*` — the split is deliberate.
- Do not remove the `(direct)` buttons; they are the escape hatch when the
  assumed state and reality disagree.
- Do not "fix" the ~430 ms scheduler warnings on gate transmission; they are
  the expected cost of `non_blocking: false`.

---

## 14. Remaining stages

**Stage 7 — RFID.** Not started, hardware not bought. Pins 14, 21, 42 reserved.
External NFC fobs, 3D-printed, with NTAG213 stickers inside.

Everything else is complete: ESP power, remote power, optocouplers, CC1101 with
learning, physical switches, Home Assistant, perfboard, enclosure.
