# janus-gate — handoff / context dump

Context for anyone (human or agent) picking this project up. Written after the
build session that got the device to a working state on the bench. The goal of
the next pass is **cleanup before a production flash**, not new features.

The authoritative source of truth is `janus-gate.yaml`. Where this document and
the YAML disagree, the YAML wins — but several YAML comments are known to be
stale, and fixing them is one of the cleanup tasks listed at the end.

---

## 1. What the device is

A wall-mounted controller, powered from a single USB-C supply, that drives
**two independent gate motors** and is reachable from Home Assistant over
Wi-Fi — while remaining fully functional with the network down.

It replaces two battery-powered RF remotes and a pile of separate hardware
with one box.

### Two completely different control paths

**GARAGE** — the original 3 V remote is physically inside the enclosure. Its
battery is gone; it runs off the ESP32's 3.3 V rail. The ESP32 shorts the
remote's own button contacts through optocouplers, exactly like a finger
press. The drive therefore sees a remote it was already paired with — no
protocol had to be reverse-engineered, and no gate security was bypassed.

**GATE** — the second remote is *not in our possession*. A CC1101 433 MHz
transceiver learns its raw OOK frame **on the device itself**, stores it in
NVS, and replays it. No code is baked into firmware.

### Why learning is on-device

This was a late redesign and it matters. The original plan was to capture the
gate remote's protocol at the bench and hardcode it. But the remote lives at
the gate, not here. Rather than make two trips, the device grew a learn mode
usable from a phone via the ESP's own access point.

The side effect is a better product: the device can be re-taught, moved to a
different gate, or taught an additional remote without reflashing.

---

## 2. Hardware inventory

### In the build

| Item | Notes |
|---|---|
| ESP32-S3 DevKitC-1 WROOM-1 N16R8 | 16 MB flash, 8 MB **octal** PSRAM |
| 2 × PC817 2-channel optocoupler module | 4 channels used, 4 spare modules left over |
| CC1101 433 MHz — **Ebyte E07-M1101D-SMA** | 1 used, 1 spare. SMA antenna fitted |
| 2 × ON-OFF-ON momentary toggle, DPDT | 6 pins each; only one pole wired |
| Remote 1 "Garage" | 3 V (was CR2032), 4 buttons: Stop / Open / Close / Lock |
| 100 nF + 47 µF | decoupling, one pair at the remote, one at the CC1101 |
| Perfboard 5 × 7 cm | stage 9 complete — everything is soldered |
| 3D-printed enclosure | stage 10 complete |

### Present but unused

- 2 × AMS1117 3.3 V. The DevKit's onboard regulator carries the whole load.
  These are the fallback if RF range turns out poor or the remote misbehaves
  during CC1101 transmit — a separate filtered rail for the radio.
- 1 spare CC1101, 2 spare PC817 modules.

### Not yet purchased

- RFID reader (stage 7). **PN532 is recommended over RC522** if the reader
  will sit outside the enclosure on a cable: RC522 is SPI-only, and SPI over
  more than ~20-30 cm gets flaky. PN532 speaks I²C and UART, which tolerate
  cable length far better.
- NTAG213 stickers for 3D-printed fobs.

### Related equipment, not part of the build

- **Sonoff RF Bridge R2, white case** — already installed in Home Assistant and
  already cloning the gate remote successfully. White case means **OB38S003**,
  not the EFM8BB1 of the older black units.
  **Its firmware was never determined** (eWeLink / Tasmota / ESPHome). This was
  asked repeatedly and never answered. It matters only as a fallback: with
  alternative firmware it could dump the raw frame directly, which would make
  on-site learning unnecessary.

---

## 3. Wiring

### Garage — optocouplers

| Function | GPIO | Wire | Module | Channel |
|---|---|---|---|---|
| Stop | 4 | grey | Opto 1 | 2 |
| Open | 5 | white | Opto 1 | 1 |
| Close | 6 | yellow | Opto 2 | 1 |
| Lock | 7 | orange | Opto 2 | 2 |

GPIO → IN+ on the module, GND → IN−. The transistor side goes to the remote's
button pads: collector to the pad measured at the higher potential, emitter to
the lower. **The transistor side needs no ground connection at all** — it just
shorts two contacts and has no idea what potential they sit at.

### Physical switches

| Switch | Lever | GPIO | Action |
|---|---|---|---|
| Garage | up | 15 | `cmd_garage_open` |
| Garage | down | 16 | `cmd_garage_close` |
| Gate | up | 17 | `cmd_gate_open` |
| Gate | down | 18 | `cmd_gate_close` |

Common pin to GND, internal pullups, no external resistors. See the lever
inversion trap in section 6.

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
| 9, 10 | GND | optional |

### Power topology

Single USB-C into the DevKit. Both the remote and the CC1101 take **their own
branch straight from the DevKit's 3V3 pin** — star, not daisy chain. Ground
likewise returns to one point.

The reason: the remote is an analogue RF transmitter and the CC1101 draws
current in short spikes when transmitting. Chained on a shared wire run, each
would modulate the other's supply.

### Pins to keep clear

- 26–32 — flash
- **33–37 — octal PSRAM** (specific to the N16R8 variant)
- 0, 3, 45, 46 — strapping
- 19, 20 — USB-JTAG
- 43, 44 — UART0
- 38 or 48 — RGB LED, depends on board revision
- 14, 21, 42 — reserved for the RFID reader (stage 7)

---

## 4. Stage status

| # | Stage | Status |
|---|---|---|
| 1 | ESP32 power-up | done |
| 2 | Remote 1 powered from ESP rail | done |
| 3 | Optocouplers → remote contacts | done, all 4 channels verified |
| 4 | (merged into 3) | — |
| 5 | CC1101 + learning | implemented; **learning verified on a substitute remote only** |
| 6 | Physical switches | done, both switches wired |
| 7 | RFID | not started, hardware not bought |
| 8 | Home Assistant | not done — see section 8 |
| 9 | Soldering onto perfboard | done |
| 10 | Enclosure | done |

---

## 5. Design decisions worth preserving

These are the non-obvious choices. Changing them without understanding why
they exist will break things that currently work.

### Raw timings, not decoded protocols

Learned codes are stored as raw pulse arrays (`int32_t[128]` + a length), not
as a decoded rc_switch/Princeton code. Decoded storage would be far smaller
and simpler — but it only works if a built-in decoder recognises the protocol.
Since learning happens on site with no chance to debug, universality beat
elegance. Raw replay works for any OOK encoding, recognised or not.

### Two-frame confirmation

The receiver hears background noise constantly, and noise bursts can be long
enough to look like a real frame. Storing the first frame that arrives would
mean capturing junk before the remote is even pressed.

So the first usable frame is only held as a *candidate*. A code is stored only
when a second frame arrives matching it within tolerance (a quarter of each
pulse width plus a flat 60 µs). Real remotes repeat their frame; noise does
not.

**Operational consequence: hold the remote button down rather than tapping it.**

### `send_*` primitives vs `cmd_*` logic

`send_garage_open` etc. only pulse an optocoupler or key the radio.
`cmd_garage_open` etc. carry the direction logic and call the primitives.

Two reasons. It avoids recursion (`cmd_open` calling `send_stop`, which would
otherwise loop if `button_stop` called back into `cmd_*`). And it leaves the
raw actions reachable for debugging — exposed as the `(direct)` diagnostic
buttons, which bypass the state machine entirely.

### Anti-reverse: opposite direction while moving sends Stop

The garage drive **reverses immediately** on the opposite command, with no
stop in between. That is hard on the motor and gearbox under load, and it is
almost never what the user meant when they hit the opposite direction.

So the device tracks assumed direction and substitutes a Stop.

**The state is an assumption, never a measurement.** There is no feedback from
either drive. It drifts if someone uses the original remote, or if the door
stalls. This was accepted deliberately because the failure modes are
asymmetric:

- Stale "moving" state → one wasted press, because Stop on a stationary door
  does nothing.
- Stale "idle" state → behaves exactly as if the feature did not exist.

Neither case is worse than not having the feature, which is what makes a
cheap, unverifiable heuristic acceptable here.

`travel_time` is deliberately **longer** than real travel (45 s vs ~30 s) for
the same reason: too long costs a press, too short lets an opposite command
reverse the drive — the exact thing being prevented.

### Gate anti-reverse has a fallback the garage does not need

The garage's Stop is a hard-wired contact and always exists. The gate's Stop
is a learned RF code and **may never have been learned** — the gate remote may
not even have a Stop button.

So `cmd_gate_*` checks `len_gate_stop > 0`. If there is no Stop code, it sends
the requested direction anyway and logs a warning. Doing nothing would be
worse than the reverse this logic exists to prevent.

### Lock is dangerous and treated as such

On most drives "Lock" disables the motor and makes it ignore every remote
until unlocked. It is the one command that can lock the user out remotely.
It has a distinct icon, a warning comment, and should be hidden or
confirmation-gated in Home Assistant.

---

## 6. Traps discovered the hard way

Each of these cost real debugging time. They are the most valuable part of
this document.

### Remote battery contacts shorting

Symptom: ESP32 died the instant the remote was connected — a brief flash, then
nothing. Cause: the metal battery tabs inside the remote were squashed together
and shorting. USB overcurrent protection was doing its job.

**The PCB was fine. Diagnosis only worked because the remote and the board
were measured separately.** That split-and-measure approach is the general
lesson here.

Since the battery is gone the tabs are useless — remove, cut or insulate them,
or the fault returns once the enclosure is closed.

### Optocoupler module jumpers must stay REMOVED

Symptom: a channel conducted permanently, holding a remote button down, which
locked up the remote's encoder and made *all* buttons stop working. The remote
looked broken.

The per-channel jumpers on these PC817 modules select the trigger polarity.
Fitted, the channels are permanently active. This must be recorded because
nothing on the board says so.

### Toggle lever inversion

A toggle lever pivots, so pushing it **up** closes the contact on the **lower**
side of the body. GPIO 15/16 and 17/18 are assigned by *lever position as the
user experiences it*, which is the opposite of the terminal position.

This was fixed by swapping the wires, not the config. Wires are on detachable
connectors; if they are ever re-made, the inversion applies again.

### Ebyte CC1101 pinout differs from the generic module

Generic CC1101 breakouts put GDO2 on pin 7 and MISO on pin 8.
**The Ebyte E07-M1101D has them the other way round.** Wiring from a generic
pinout diagram gives a dead SPI bus.

Reliable orientation trick: **pin 1 is GND and has continuity with the SMA
connector shell.** Beep from the shell to each pin; the one that beeps is pin 1.

### Wrong USB port on the DevKitC-1

The board has two USB-C sockets: one through a CP2102N bridge (marked UART),
one native to the ESP32-S3 (marked USB). First flash **must** use the UART
port. The native port is unreliable until working firmware is present.

Tell: if Windows shows the port as a generic "USB Serial Device", that is the
native port. The UART port shows up as a Silicon Labs CP210x bridge.

### `dump: all` floods the log with garbage

`dump: all` also runs the *infrared* decoders. Pronto and Beo4 in particular
will happily "recognise" pure RF noise and emit megabytes of hex. Raw replay
needs no decoder at all.

`dump: none` is **not valid** — there is no dumper by that name. To disable
dumping, omit the `dump:` line entirely. `on_raw` fires regardless of `dump`.

### Over-filtering killed reception entirely — twice

This is the big one, and it was my error both times.

Attempting to reduce the noise floor, `filter_bandwidth` was dropped to 58 kHz
and `remote_receiver.filter` raised to 250 µs. **Reception stopped completely.**

- Cheap remotes use SAW resonators that can be off nominal by hundreds of kHz.
  A 58 kHz window simply misses them. Wide bandwidth forgives that detuning.
- The remote's short pulses measured ~275–300 µs, i.e. *below* a 250 µs filter
  once jitter is accounted for. Every bit was being discarded as a spike.

**The correct answer was not to fight noise at the radio layer at all.** The
two-frame confirmation already rejects noise in software. Let the receiver
hear everything and filter in logic.

---

## 7. Settings that are load-bearing — do not "improve" these

These values were found empirically and verified working. Several of them look
wrong or wasteful and are not.

```yaml
cc1101:
  filter_bandwidth: 200kHz    # NOT 58kHz — narrow bandwidth kills reception
  rx_attenuation: 0dB         # noise is handled in software, not here

remote_receiver:
  filter: 100us               # NOT 250us — the remote's pulses are ~275us
  idle: 6ms

api:
  reboot_timeout: 0s          # keep this even after HA is connected
```

`pulse: 400ms` for the garage optocouplers is verified working on this drive.

On `reboot_timeout`: a YAML comment currently says to remove it once Home
Assistant is connected. **That advice is wrong for this device** and should be
deleted. The default behaviour reboots the ESP every 15 minutes while HA is
unreachable — meaning a router or server outage turns a gate controller into
something that reboots in a loop.

---

## 8. Open questions and unverified assumptions

Ordered by how much damage each could cause.

### Never tested: transmission end-to-end

Learning was verified against a substitute remote of the same type. **A learned
code has never been replayed to an actual receiver and seen to work.** The
transmit path (GDO0 → GPIO 9 → `remote_transmitter`) is therefore unproven.

Relevant known issue: setting `gdo0_pin` inside the `cc1101:` block re-routes
the pad away from the RMT peripheral and silently kills transmit — the chip
enters TX but nothing reaches the air. **The config deliberately omits
`gdo0_pin`. Do not add it.**

### Unverified: NVS persistence across power loss

Learned codes are stored in `int32_t[128]` globals with `restore_value: true`.
This is a non-standard use of ESPHome's preference system. A reboot test is not
sufficient — **pull the USB cable physically**, wait, reconnect, and check that
"Gate close pulses" still reads non-zero.

If it does not survive, the fallback design is to store a decoded rc_switch
code (protocol + code + bit length) instead, which is tiny and definitely
persists — at the cost of only working for recognised protocols.

### Unknown: gate remote frequency

`rf_frequency` is set to 433.92 MHz, which covers most remotes. Some gate
remotes use 433.42 MHz, and **this cannot be changed without reflashing**.

Mitigation already in place: `ota: platform: web_server` is enabled, so a
second firmware built at 433.42 MHz can be kept on a phone as
`firmware.ota.bin` and uploaded through the browser on site.

### Unknown: gate drive behaviour

Everything in `cmd_gate_*` assumes the gate reverses instantly on the opposite
command, copied from the garage. If the gate stops by itself, the logic adds
nothing and the switches and buttons can be pointed back at `send_gate_*`.

### Unknown: gate travel time

`gate_travel_time: 45s` is a guess copied from the garage. Measure it.

### Unknown: does the gate remote have a Stop button

If not, the Stop slot stays empty and `cmd_gate_*` falls back to sending the
direction. Consider teaching something else useful into that slot.

### Untested: the on-site access path

When away from the home network the ESP serves its own AP ("Janus Gate
Fallback"); a phone joins it and browses to `192.168.4.1`. **This has not been
rehearsed.** Test it by temporarily setting a wrong SSID before travelling — it
is the only route to the device in the field.

### Unresolved: where Home Assistant lives

If HA is on the same LAN as the device, the native ESPHome integration works
with no config changes: Settings → Devices & Services → ESPHome, host
`janus-gate.local` or the IP, port 6053, then paste `api_key` from
`secrets.yaml`.

**If HA is in a different location from the gate, this does not work over the
internet** and the project needs either a VPN or a switch to MQTT. That is a
substantially different piece of work and it was never established which
situation applies.

### Security posture, stated plainly

`web_server` has HTTP basic auth. ESP32 does not do TLS here, so credentials
cross the network in the clear. This deters casual access on a trusted LAN; it
is not protection against a determined attacker already inside the network.

The bigger exposure is the fallback AP, which is visible from outside the
building whenever the main Wi-Fi is unreachable. `fallback_password` should be
long and random, not memorable.

And the physical switches work with no authentication at all. That is
intentional and is the entire point of the device — but it means security
ultimately reduces to who can reach the box on the wall.

---

## 9. Cleanup tasks for this pass

The config works. These are hygiene items before a production flash.

### Stale comments that contradict the code

1. The `cc1101:` block comment claims `filter_bandwidth` "is at the minimum"
   and that "58 kHz is plenty". The actual value is **200 kHz**, deliberately.
   Rewrite the comment to explain *why* wide bandwidth is correct here
   (SAW detuning, noise handled in software).
2. The `remote_receiver:` block comment discusses `dump: raw` and
   `filter: 250us`. Neither is in the config any more — `dump` is omitted and
   `filter` is 100 µs. Rewrite to match, and keep the warning about `dump: all`
   running IR decoders.
3. The `api:` comment says to remove `reboot_timeout: 0s` once HA is connected.
   Delete that advice and replace it with the reason it must stay.
4. Header `STATUS:` lines are out of date — stages 6, 9 and 10 are complete.

### Logging that was lost

The `on_raw` lambda currently does `if (n < 24 || n > 128) return;` silently.
An earlier revision logged "Too short: N pulses" and "Too long: N pulses -
frames may be merging". **The "too long" warning in particular is diagnostic
gold on site**: it is the signal that `idle` is too short and the remote's
repeated frames are being merged into one oversized block, which would make
learning fail while the radio is working perfectly.

Consider restoring both, ideally at `ESP_LOGD`/`ESP_LOGW`.

### Worth adding

5. **`cover:` template entity.** Open, Close and Stop exist for both drives,
   which is exactly the `cover` shape. In HA that becomes one tile with arrows
   instead of a scattering of buttons, and it works with voice assistants and
   automations. It needs no knowledge of actual position.
6. **Hide or confirmation-gate the Lock entity** once HA is connected.
7. **README.md.** The repo is going public. The traps in section 6 are more
   interesting than the architecture and most projects never document them.
8. Verify `secrets.yaml.example` (or the template) lists all six secrets now in
   use: `wifi_ssid`, `wifi_password`, `api_key`, `ota_password`,
   `fallback_password`, `web_username`, `web_password`.
9. `sniffer.yaml` — a standalone receive-only config from the protocol
   investigation. Either keep it with a comment explaining what it is for, or
   delete it. Right now it is unexplained clutter.

### Do not do

- Do not "optimise" the radio settings in section 7.
- Do not add `gdo0_pin` to the `cc1101:` block.
- Do not consolidate `send_*` and `cmd_*` — the split is deliberate.
- Do not remove the `(direct)` diagnostic buttons; they are the escape hatch
  when the assumed state and reality disagree.

---

## 10. Operating procedure — learning a gate code on site

1. Power the device at the gate. Away from the home network it will serve
   "Janus Gate Fallback".
2. Join that AP with a phone, browse to `192.168.4.1`, log in.
3. Press "Learn gate open". "Learn mode" shows `waiting for OPEN`.
4. **Hold** the matching button on the original remote, close to the antenna.
   Two matching frames are needed, so a tap may not be enough.
5. Watch "Learn mode" go to `candidate held`, then back to `idle`.
6. Check "Gate open pulses". A plausible fixed-code frame is roughly **25–100
   pulses**; around 50 is typical for a 24-bit frame (2 pulses per bit plus
   sync). A value of 2 or 4 means noise was captured — repeat.
7. Repeat for close and stop.
8. Test with the "Gate open (direct)" button, which bypasses the direction
   logic.

"Candidate pulses" is a live view of what is waiting for confirmation. If it
keeps jumping between unrelated values, the radio is chewing on noise rather
than hearing the remote.

Before travelling: press **"Forget all gate codes"**. Whatever is currently
stored was learned from a bench substitute, not from the real gate remote.